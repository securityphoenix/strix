# RUNTIME_FLOWS

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/CALL_GRAPH.md
- /docs/architecture/SERVICE_TOPOLOGY.md
- /docs/architecture/RUNTIME_DEPENDENCY_TREE.md
- /docs/ai/AGENT_WORKFLOWS.md

Read this when:
- Tracing how a scan executes step-by-step
- Debugging unexpected scan behavior
- Understanding resume/interrupt flows

Purpose:
- Documents top runtime flows for fresh scans, resume, CI/CD mode, and multi-agent spawning

Scope:
- Runtime behavior; not configuration or data persistence details

---

## Flow 1: Fresh Scan (Non-Interactive / CI Mode)

```
strix -n --target https://example.com
         │
         ▼
main() [interface/main.py]
  1. configure_dependency_logging()
  2. parse_arguments() → infer_target_type(), assign_workspace_subdirs()
  3. check_docker_installed()
  4. pull_docker_image() → docker.api.pull() if not locally present
  5. validate_environment() → check STRIX_LLM set
  6. asyncio.run(warm_up_llm()) → test LLM connectivity
  7. persist_current() → write env vars to ~/.strix/cli-config.json
  8. generate_run_name() → e.g. "web-application-abc123"
  9. clone_repository() → if target is git URL
 10. collect_local_sources() → resolve host paths for container mounts
 11. resolve_diff_scope_context() → compute diff scope for quick/CI mode
 12. _persist_run_record() → write strix_runs/<run>/run.json (status=running)
 13. posthog.start(), scarf.start()
 14. asyncio.run(run_cli(args))  [non-interactive path]
         │
         ▼
run_cli() [interface/cli.py]
  1. Build scan_config dict
  2. Instantiate ReportState, hydrate_from_run_dir()
  3. set_global_report_state()
  4. Register signal handlers (SIGINT, SIGTERM, SIGHUP)
  5. Spin Rich Live status thread
  6. await run_strix_scan(scan_config, image, local_sources)
         │
         ▼
run_strix_scan() [core/runner.py]
  1. Create run_dir, state_dir
  2. setup_scan_logging() → per-run log file
  3. load_settings(), configure_sdk_model_defaults()
  4. resolve model name
  5. Instantiate AgentCoordinator
  6. hydrate_notes_from_disk(), hydrate_todos_from_disk()
  7. await session_manager.create_or_reuse()
       → get_backend("docker") → StrixDockerSandboxClient.create() → session.start()
       → bootstrap_caido() → guest login + project select
  8. Build RunConfig (model, model_settings, sandbox)
  9. build_root_task() → assemble targets + instructions text
 10. build_strix_agent("strix", is_root=True, ...)
       → render_system_prompt() → Jinja2 render of system_prompt.jinja
            + skill Markdown files loaded and injected
       → SandboxAgent(tools=[think, web_search, proxy tools, notes, todos,
                              create_vulnerability_report, agents_graph tools,
                              finish_scan], capabilities=[Filesystem, Shell])
 11. coordinator.register(root_id, "strix", parent_id=None, task=root_task)
 12. make_child_factory() → closure capturing scan-level params
 13. spawn_child_agent closure assembled
 14. context dict assembled: coordinator, sandbox_session, caido_client,
     agent_id, parent_id, interactive, spawn_child_agent
 15. open_agent_session(root_id, agents_db)
 16. await run_agent_loop(root_agent, initial_input=root_task, ...)
         │
         ▼
run_agent_loop() [core/execution.py]
  → _run_noninteractive_until_lifecycle()
       LOOP:
         _run_cycle() → Runner.run_streamed() → LLM calls → tool calls
         if status != "running": return
         else: retry with tool-required message (up to max_turns times)
         │
         ▼ (inside _run_cycle)
       Runner.run_streamed() streams events
         LLM generates → tool calls → Filesystem/Shell execute in container
         Tools: think, web_search, proxy tools, create_vulnerability_report,
                create_agent (→ spawns child agents as asyncio tasks),
                wait_for_message, finish_scan
         On finish_scan: _finish_tool_use_behavior returns is_final_output=True
         → run exits, status set by lifecycle tool
         │
         ▼
Back in run_strix_scan() finally block:
  Close sessions
  coordinator._maybe_snapshot() → write agents.json
  session_manager.cleanup() → client.delete(session) → container removed
  teardown_logging()
         │
         ▼
Back in run_cli() / main():
  display_completion_message()
  exit(2) if vulnerabilities found (non-interactive mode)
```

---

## Flow 2: Resume Scan

```
strix --resume <run-name>
  1. _load_resume_state() → read strix_runs/<run>/run.json
     → restore targets_info, instruction, scan_mode, local_sources, diff_scope
  2. Verify cloned repo paths still exist
  3. Pull Docker image if needed (same as fresh)
  4. run_strix_scan() with is_resume=True
         │
         ▼
run_strix_scan() (resume path):
  1. Read agents.json → coordinator.restore(snap)
  2. Find root_id (parent_of == None)
  3. session_manager.create_or_reuse() → new container (fresh sandbox)
     NOTE: agents.db carries conversation history; /workspace state is LOST
  4. Build RunConfig, rebuild root_agent (same instructions)
  5. respawn_subagents() → for each non-terminal child in snapshot:
       _start_child_runner() → asyncio.create_task(run_agent_loop(...))
       child input = [] (replays from agents.db session history)
  6. If resume_instruction provided: coordinator.send(root_id, instruction)
  7. run_agent_loop(root_agent, initial_input=[], start_parked=...)
     initial_input=[] causes SDK to replay from session history in agents.db
```

---

## Flow 3: Child Agent Spawn

```
Root agent calls create_agent(name, task, skills=[...])
         │
         ▼
tools/agents_graph/tools.py:create_agent()
  1. validate_requested_skills(skill_list)
  2. Capture parent_history from ctx.turn_input (if inherit_context=True)
  3. await spawner(parent_ctx, name, task, skills, parent_history)
         │
         ▼
core/execution.py:spawn_child_agent()
  1. child_id = uuid4().hex[:8]
  2. factory(name=name, skills=skills) → build_strix_agent(is_root=False, ...)
  3. coordinator.register(child_id, name, parent_id, task, skills)
  4. _start_child_runner():
       open_agent_session(child_id, agents_db)
       child_ctx = dict(parent_ctx); child_ctx["agent_id"] = child_id
       asyncio.create_task(run_agent_loop(child_agent, ...))
       coordinator.attach_runtime(child_id, task=task_handle)
  5. Returns immediately to parent: {success, agent_id, name, parent_id}
Parent continues running while child runs in parallel asyncio task.
```

---

## Flow 4: Vulnerability Report Filing

```
Agent calls create_vulnerability_report(title, ..., cvss_breakdown, ...)
         │
         ▼
tools/reporting/tool.py:create_vulnerability_report()
  1. Extract agent_id, agent_name from context
  2. _do_create():
     a. Validate all required fields
     b. Validate CVSS breakdown (8 metrics)
     c. _normalize_code_locations() + validate
     d. Validate CVE/CWE format
     e. _calculate_cvss() → cvss3 library → score + severity + vector
     f. get_global_report_state()
     g. check_duplicate(candidate, existing) → LLM call via MultiProvider
        → returns {is_duplicate, duplicate_id, confidence, reason}
     h. If duplicate: return error (do not re-file)
     i. report_state.add_vulnerability_report(...)
        → posthog.finding(), scarf.finding()
        → vulnerability_found_callback(report) → live UI display
        → save_run_data() → write artifacts to disk
  3. Return JSON result to agent
```

---

## Flow 5: Interactive TUI Mode

```
strix --target https://example.com  (no -n flag)
  → asyncio.run(run_tui(args))
  [TUI app manages the display; run_strix_scan() called with interactive=True]
  
  In run_agent_loop (interactive mode):
    - Each LLM turn runs one _run_cycle()
    - After turn: status set to "waiting" (not crashed)
    - wait_for_message tool parks the agent until new input arrives
    - User can send messages via TUI → coordinator.send() → wake agent
    - Agent can be interrupted between turns via stream.cancel()
```

---

## Known Gaps / Uncertainties

- TUI event handling details (keyboard shortcuts, message dispatch) not fully analyzed.
- Exact behavior when multiple child agents all try to call `finish_scan` simultaneously (should not happen — only root calls `finish_scan`).
- Error recovery path when `agents.db` is present but `agents.json` is missing (documented as `RuntimeError` in runner.py but recovery path not clear).
