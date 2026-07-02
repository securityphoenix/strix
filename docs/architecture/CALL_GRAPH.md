# CALL_GRAPH

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/RUNTIME_FLOWS.md
- /docs/architecture/DEPENDENCY_GRAPH.md

Read this when:
- Tracing function-level call paths from entry points
- Debugging a crash or unexpected exit path
- Understanding which functions are on the hot path

Purpose:
- Key call paths from entry points to core modules

Scope:
- Important paths only; not exhaustive function list

---

## Main Entry Point → Scan Start

```mermaid
graph TD
    main["main()"] --> parse_arguments
    main --> check_docker_installed
    main --> pull_docker_image
    main --> validate_environment
    main --> warm_up_llm
    main --> persist_current
    main --> run_cli_or_tui{"non_interactive?"}
    run_cli_or_tui -->|yes| run_cli
    run_cli_or_tui -->|no| run_tui
    run_cli --> run_strix_scan
    run_tui --> run_strix_scan
```

## run_strix_scan → Agent Execution

```mermaid
graph TD
    run_strix_scan --> setup_scan_logging
    run_strix_scan --> load_settings
    run_strix_scan --> coordinator_init["AgentCoordinator()"]
    run_strix_scan --> hydrate_state["hydrate_notes/todos_from_disk"]
    run_strix_scan --> session_create["session_manager.create_or_reuse()"]
    session_create --> get_backend
    get_backend --> _docker_backend
    _docker_backend --> StrixDockerSandboxClient_create["StrixDockerSandboxClient.create()"]
    _docker_backend --> session_start["session.start()"]
    session_create --> bootstrap_caido
    bootstrap_caido --> _login_as_guest
    bootstrap_caido --> client_connect["Client.connect()"]
    bootstrap_caido --> project_create["client.project.create()"]

    run_strix_scan --> build_strix_agent
    build_strix_agent --> render_system_prompt
    render_system_prompt --> _resolve_skills
    render_system_prompt --> load_skills
    render_system_prompt --> jinja_render["Jinja2 env.get_template().render()"]

    run_strix_scan --> run_agent_loop
    run_agent_loop --> _run_noninteractive_until_lifecycle
    _run_noninteractive_until_lifecycle --> _run_cycle
    _run_cycle --> Runner_run_streamed["Runner.run_streamed()"]
    _run_cycle --> stream_events["async for event in stream.stream_events()"]
    _run_cycle --> _settle_run_result
```

## Tool Call → Vulnerability Filed

```mermaid
graph TD
    LLM_calls_tool["LLM calls create_vulnerability_report"] --> _do_create
    _do_create --> _validate_fields
    _do_create --> _calculate_cvss["_calculate_cvss() → cvss3 lib"]
    _do_create --> get_global_report_state
    _do_create --> check_duplicate
    check_duplicate --> MultiProvider_get_response["MultiProvider().get_model().get_response()"]
    check_duplicate --> _parse_dedupe_response
    _do_create --> report_state_add["report_state.add_vulnerability_report()"]
    report_state_add --> posthog_finding["posthog.finding()"]
    report_state_add --> vulnerability_found_callback["vulnerability_found_callback(report)"]
    report_state_add --> save_run_data
    save_run_data --> _save_artifacts
    _save_artifacts --> write_executive_report
    _save_artifacts --> write_vulnerabilities
    _save_artifacts --> write_run_record
```

## Tool Call → Child Agent Spawned

```mermaid
graph TD
    LLM_calls_create_agent["LLM calls create_agent()"] --> validate_requested_skills
    LLM_calls_create_agent --> spawner_call["spawner(parent_ctx, name, task, skills, history)"]
    spawner_call --> spawn_child_agent_fn["execution.spawn_child_agent()"]
    spawn_child_agent_fn --> coordinator_register["coordinator.register(child_id, ...)"]
    spawn_child_agent_fn --> _start_child_runner
    _start_child_runner --> open_agent_session["open_agent_session(child_id, agents_db)"]
    _start_child_runner --> asyncio_create_task["asyncio.create_task(run_agent_loop(...))"]
    _start_child_runner --> coordinator_attach_runtime["coordinator.attach_runtime(child_id, task=task_handle)"]
```

## Scan Completion Path

```mermaid
graph TD
    LLM_calls_finish_scan["LLM calls finish_scan()"] --> update_scan_final_fields
    update_scan_final_fields --> posthog_end["posthog.end()"]
    update_scan_final_fields --> save_run_data_complete["save_run_data(mark_complete=True)"]
    LLM_calls_finish_scan --> _finish_tool_use_behavior
    _finish_tool_use_behavior --> ToolsToFinalOutputResult_final["is_final_output=True"]
    ToolsToFinalOutputResult_final --> run_agent_loop_returns["run_agent_loop() returns"]
    run_agent_loop_returns --> run_strix_scan_finally["run_strix_scan() finally block"]
    run_strix_scan_finally --> coordinator_snapshot["coordinator._maybe_snapshot()"]
    run_strix_scan_finally --> session_manager_cleanup["session_manager.cleanup()"]
    session_manager_cleanup --> caido_aclose["caido_client.aclose()"]
    session_manager_cleanup --> client_delete["client.delete(session)"]
```

---

## Known Gaps / Uncertainties

- Interactive TUI event dispatch not traced.
- Error recovery path for `image_strips >= 3` in `_run_cycle` not fully traced (falls through to raise after strip exhaustion).
- Exactly when `coordinator._maybe_snapshot()` is called vs. `run_agent_loop` finishing is not fully sequenced (it's called at multiple points inside the coordinator methods too).
