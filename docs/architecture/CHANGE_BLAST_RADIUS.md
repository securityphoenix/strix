# CHANGE_BLAST_RADIUS

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/DEPENDENCY_GRAPH.md
- /docs/architecture/DATA_CONTRACTS.md
- /docs/development/SAFE_CHANGE_ZONES.md

Read this when:
- Assessing risk before modifying a module
- Planning a refactor
- Reviewing a PR that touches high-risk files

Purpose:
- Documents which modules have the highest change impact and what breaks

Scope:
- Source-level blast radius; not deployment risk

---

## Highest Impact — Touch With Extreme Care

### `strix/agents/prompts/system_prompt.jinja`
**Blast radius**: ALL agent behavior.
- Changes here affect every scan, every agent, in every mode.
- The Jinja template controls tool usage discipline, output format, scope enforcement, multi-agent coordination, and refusal avoidance.
- Scope injection block (`system_prompt_context.authorized_targets`) is **security-critical** — removing it removes the scope enforcement mechanism.
- Interactive vs autonomous behavior branches are controlled here.
- **Never** change without testing both interactive and non-interactive modes.
- **Direct breaks**: Wrong tool usage instructions → agents produce text turns → `_run_noninteractive_until_lifecycle` hits recovery limit → `MaxTurnsExceeded`.

### `strix/core/execution.py`
**Blast radius**: Agent execution loop, resume, spawning, error recovery.
- `run_agent_loop` is the central loop for every agent (root and child).
- `_run_noninteractive_until_lifecycle` implements the "tool-required" recovery; changing the limit or message breaks non-interactive scans.
- Image-strip retry logic (handles 400/404/422 API responses) is subtle; removing it causes crashes when LLMs reject large image inputs.
- `spawn_child_agent` → `_start_child_runner` → `asyncio.create_task()` — changing task management here can cause zombie agents or missed cancellation.
- `respawn_subagents` is the resume path; bugs here break all resume functionality.

### `strix/core/agents.py` — AgentCoordinator
**Blast radius**: Agent graph state, messages, snapshots, resume.
- `_maybe_snapshot()` is called atomically on every state change. Removing atomicity → corrupt `agents.json` → broken resume.
- `send()` injects messages into agent SDK sessions; wrong implementation → agents never wake up or receive corrupt state.
- `_lock` (asyncio.Lock) protects all mutable state; removing it → data races under concurrent child agents.

### `strix/report/state.py` — ReportState
**Blast radius**: All scan artifacts (vulnerabilities, run record, reports).
- Global singleton set in `interface/cli.py`. Any tool that calls `get_global_report_state()` depends on this.
- `add_vulnerability_report()` is the only way to file vulns — changing its `report_id` allocation (`vuln-{N:04d}`) breaks the dedup ID lookup on resume.
- `save_run_data()` is called on every vulnerability filing — making it blocking or throwing would break agent execution.
- `vulnerability_found_callback` is a Callable set by the interface layer — must remain thread-safe (called from async context, consumed by Rich Live).

---

## Medium Impact — Change Carefully

### `strix/agents/factory.py`
**Blast radius**: Agent capabilities, tool list, lifecycle tool detection.
- `_BASE_TOOLS` tuple defines exactly which tools every agent gets. Removing a tool breaks any agent behavior relying on it.
- `_lifecycle_tool_completed` and `_wait_tool_parked` define the exit conditions for the agent loop. Changing the `agent_completed` / `scan_completed` key names breaks the loop termination.
- `chat_completions_tools` branching: wrong detection → agents crash on providers that don't support Responses-format custom tools.

### `strix/config/loader.py`
**Blast radius**: All settings, config persistence.
- `_cached` singleton reset only by `apply_config_override()`. If settings are needed before `load_settings()` is called, they'll get wrong defaults.
- `persist_current()` writes to `~/.strix/cli-config.json` with 0o600 perms. Changing which keys are written silently drops user config on next load.

### `strix/tools/reporting/tool.py`
**Blast radius**: Vulnerability report contract, CVSS calculation.
- All required field validation is here. Adding a new required field breaks existing agents until the prompt is updated.
- CVSS calculation failure falls back to score=7.5/severity=high — wrong fallback could misclassify all vulns when cvss lib fails.

### `strix/runtime/session_manager.py`
**Blast radius**: Container lifecycle, Caido connection.
- `_SESSION_CACHE` is a module-level dict. Memory leak if `cleanup()` is never called.
- `cleanup()` is best-effort (logs and swallows errors). Stranded containers are a real risk on crash.
- `bootstrap_caido()` has a 10-attempt retry loop. Reducing attempts → Caido readiness race condition.

---

## Lower Impact — Safer to Change

### `strix/interface/utils.py`
- Target type inference, diff scope resolution, run name generation.
- Changes to `infer_target_type()` affect what targets are recognized.
- Changes to `resolve_diff_scope_context()` affect CI/PR diff-scoped behavior.

### `strix/skills/` (Markdown files)
- New skill files: safe to add (auto-discovered).
- Modifying existing skill content: affects agent behavior for scans using that skill.
- Changing skill frontmatter stripping regex in `skills/__init__.py`: may expose frontmatter to LLM.

### `strix/telemetry/`
- PostHog/Scarf are fire-and-forget with broad exception swallowing.
- Changing event names or properties: no functional impact, breaks analytics.

### `strix/report/writer.py`
- File I/O for run artifacts. Adding new output files is safe.
- Changing `vulnerabilities.json` format: breaks `hydrate_from_run_dir()` on resume.

### `strix/tools/notes/tools.py`, `strix/tools/todo/tools.py`
- In-memory + disk state for agent scratch-pad.
- Changing disk format: breaks `hydrate_notes_from_disk()` / `hydrate_todos_from_disk()` on resume.

---

## Schema Risk Summary

| Schema | Risk Level | Why |
|--------|-----------|-----|
| `vulnerabilities.json` format | HIGH | Resume dedup depends on `id` field |
| `agents.json` format | HIGH | Resume depends on all coordinator fields |
| `agents.db` SQLite schema | HIGH | Owned by SDK; upgrades may break resume |
| `run.json` format | MEDIUM | Resume reads `targets_info`, `instruction` |
| `create_vulnerability_report` tool args | MEDIUM | Breaking = silent drop of vulns |
| `~/.strix/cli-config.json` format | LOW | Best-effort; falls back to env defaults |

---

## Known Gaps / Uncertainties

- Impact of changing `MAX_TURNS` default (currently 500) on existing scans is unknown.
- Whether the `openai-agents` SDK version upgrade would break `agents.db` schema (SQLite migration) is unknown and unmitigated.
