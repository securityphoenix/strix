# MODULE_OWNERSHIP

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/REPOSITORY_MAP.md
- /docs/architecture/CHANGE_BLAST_RADIUS.md

Read this when:
- Looking for which module is responsible for a capability
- Deciding where new code should live

Purpose:
- Maps functional concerns to owning modules

Scope:
- Source ownership only

---

## Ownership Map

| Concern | Owning Module(s) |
|---------|-----------------|
| CLI argument parsing | `strix/interface/main.py` |
| Pre-flight checks (docker, env, LLM warmup) | `strix/interface/main.py` |
| Non-interactive UI (Rich console) | `strix/interface/cli.py` |
| Interactive TUI | `strix/interface/tui/` |
| Target type inference | `strix/interface/utils.py` |
| Diff scope resolution | `strix/interface/utils.py` |
| Scan orchestration (top-level) | `strix/core/runner.py` |
| Agent execution loop | `strix/core/execution.py` |
| Agent graph state + messaging | `strix/core/agents.py` |
| Scan input construction (task, scope) | `strix/core/inputs.py` |
| Agent session (SQLite) management | `strix/core/sessions.py` |
| Token usage tracking (hooks) | `strix/core/hooks.py` |
| Run directory path conventions | `strix/core/paths.py` |
| Agent construction + tool assembly | `strix/agents/factory.py` |
| System prompt rendering (Jinja2) | `strix/agents/prompt.py` |
| System prompt template (all behavior) | `strix/agents/prompts/system_prompt.jinja` |
| Skill loading and validation | `strix/skills/__init__.py` |
| Skill content (knowledge) | `strix/skills/**/*.md` |
| think tool (agent reasoning scratch-pad) | `strix/tools/thinking/tool.py` |
| load_skill tool (inline skill loading) | `strix/tools/load_skill/tool.py` |
| Note management tools | `strix/tools/notes/tools.py` |
| Todo management tools | `strix/tools/todo/tools.py` |
| Web search tool | `strix/tools/web_search/tool.py` |
| Proxy tools (Caido API) | `strix/tools/proxy/tools.py` |
| Caido SDK wrapper | `strix/tools/proxy/caido_api.py` |
| Vulnerability report filing | `strix/tools/reporting/tool.py` |
| Multi-agent graph tools | `strix/tools/agents_graph/tools.py` |
| Root agent lifecycle (finish_scan) | `strix/tools/finish/tool.py` |
| Settings model | `strix/config/settings.py` |
| Settings loading + persistence | `strix/config/loader.py` |
| Model normalization + SDK defaults | `strix/config/models.py` |
| Container backend registry | `strix/runtime/backends.py` |
| Container lifecycle + session cache | `strix/runtime/session_manager.py` |
| Docker client (capabilities injection) | `strix/runtime/docker_client.py` |
| Caido bootstrap (guest login, project) | `strix/runtime/caido_bootstrap.py` |
| Scan artifact state (global) | `strix/report/state.py` |
| Artifact disk I/O | `strix/report/writer.py` |
| Vulnerability deduplication (LLM) | `strix/report/dedupe.py` |
| LLM usage accounting | `strix/report/usage.py` |
| PostHog telemetry | `strix/telemetry/posthog.py` |
| Scarf telemetry | `strix/telemetry/scarf.py` |
| Resource path resolution (bundled assets) | `strix/utils/resource_paths.py` |

---

## Known Gaps / Uncertainties

- `strix/interface/tui/` internal ownership (renderers, keybindings, etc.) not fully mapped.
- `strix/config/models.py` exact responsibilities not fully analyzed.
