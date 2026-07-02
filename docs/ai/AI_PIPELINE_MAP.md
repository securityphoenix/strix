# AI_PIPELINE_MAP

Parent document: /CLAUDE.md
Related documents:
- /docs/ai/LLM_ARCHITECTURE.md
- /docs/ai/PROMPTS.md
- /docs/ai/AGENT_WORKFLOWS.md
- /docs/architecture/RUNTIME_FLOWS.md

Read this when:
- Tracing the full AI pipeline end-to-end
- Debugging model behavior or output issues
- Understanding failure points in the AI path

Purpose:
- End-to-end AI pipeline from context assembly to persistence

Scope:
- Main scan pipeline; not warm-up or telemetry

---

## End-to-End AI Pipeline

```
1. CONTEXT ASSEMBLY
   ─────────────────────────────────────────────────────────────
   build_scope_context(scan_config) → authorized_targets dict
   build_root_task(scan_config) → targets + diff_scope + instructions text
   
   render_system_prompt(skills, scan_mode, is_whitebox, is_root, interactive,
                        system_prompt_context)
     ├─ _resolve_skills() → ordered deduped skill list
     ├─ load_skills(skill_names) → {skill_name: markdown_content}
     └─ Jinja2: system_prompt.jinja rendered with all vars → system prompt string
   
   SandboxAgent(instructions=system_prompt, tools=[...])

2. PROMPT GENERATION (per turn)
   ─────────────────────────────────────────────────────────────
   Runner.run_streamed() assembles:
     - system: pre-rendered system_prompt (constant across turns)
     - messages: full conversation from SQLiteSession (agents.db)
       + initial_input (first turn only, or [] for resume)
     - tools: JSON schemas of all registered FunctionTools + CustomTools
     - model_settings: tool_choice, reasoning, retry, include_usage

3. MODEL CALL
   ─────────────────────────────────────────────────────────────
   MultiProvider().get_model(resolved_model).get_response(...)
     via LiteLLM → target LLM provider API
     streaming: yes (streamed events)
     timeout: LLM_TIMEOUT (default 300s) per warm-up; per-turn timeout from SDK

4. RESPONSE PARSING
   ─────────────────────────────────────────────────────────────
   SDK streams RunStreamedResult events:
     - tool call events → invoke tool functions
     - message output events → accumulate final output
   
   Tool results injected back into session → next turn context
   
   _finish_tool_use_behavior() checks tool results:
     - finish_scan / agent_finish succeeded → is_final_output=True → run exits
     - wait_for_message parked (interactive) → is_final_output=True → run yields

5. TOOL EXECUTION (in-turn)
   ─────────────────────────────────────────────────────────────
   Host-side tools: execute Python functions in host process
     - think: no-op scratch-pad
     - web_search: HTTP to Perplexity API
     - proxy tools: HTTP to Caido SDK (host → container port)
     - notes/todos: in-memory + disk state
     - create_vulnerability_report: validation + LLM dedupe + ReportState
     - agents_graph tools: coordinator state mutation + asyncio task creation
   
   Sandbox tools: execute inside Docker container via SDK exec
     - Shell tools (exec_command, write_stdin, etc.)
     - Filesystem tools (read_file, write_file, apply_patch, etc.)
     Tool results returned to host → appended to SDK session → next turn

6. VALIDATION (for create_vulnerability_report only)
   ─────────────────────────────────────────────────────────────
   Field validation → CVSS calculation → deduplication LLM call
   
   Deduplication pipeline:
     a. Prepare candidate + existing reports (truncate to 8000 chars/field)
     b. MultiProvider().get_model().get_response(DEDUPE_SYSTEM_PROMPT, user_msg)
        model_settings: ModelSettings(retry=DEFAULT_MODEL_RETRY, include_usage=True)
     c. _parse_dedupe_response() → {is_duplicate, duplicate_id, confidence, reason}
     d. record_sdk_usage(agent_id="dedupe", ...) → LLM usage ledger

7. PERSISTENCE
   ─────────────────────────────────────────────────────────────
   SDK session: agents.db (per turn, managed by SDK)
   Coordinator snapshot: agents.json (per state change, atomic)
   Vulnerability reports: vulnerabilities.json + vuln-NNNN.md (on each add_vulnerability_report)
   Run record: run.json (on each save_run_data)
   Executive report: executive_report.md (on finish_scan)

8. DOWNSTREAM CONSUMPTION
   ─────────────────────────────────────────────────────────────
   Vulnerability reports → UI callback → Rich console / TUI widget
   Executive summary → final CLI panel
   LLM usage → run.json llm_usage → telemetry posthog.end()
   Scan artifacts → strix_runs/<run>/ (user-readable)
```

---

## Failure Points

| Stage | Failure | Impact |
|-------|---------|--------|
| Context assembly | Jinja2 render fails | Empty system prompt → agent has no instructions |
| Model call | Provider API error | SDK retry → if exhausted: agent fails or crashes |
| Model call | 400/404/422 with image | Image-strip retry (up to 3 times) |
| Tool execution | Tool function raises | Error returned as string to LLM (tool errors are model-visible) |
| Dedup LLM call | Any error | Defaults to is_duplicate=False → report filed |
| Persistence | Disk write fails | Exception logged; vulnerability report may be in-memory but not persisted |
| Lifecycle tool | LLM never calls finish_scan/agent_finish | MaxTurnsExceeded after recovery attempts |

---

## Known Gaps / Uncertainties

- Token limit handling: what happens when conversation history in agents.db exceeds the model's context window is not documented. Image strips handle one case, but long text histories may hit limits silently.
- LiteLLM timeout behavior per turn (vs. warm-up timeout) not confirmed.
