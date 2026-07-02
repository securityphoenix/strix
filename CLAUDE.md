# Strix — Engineering Map

> Load this file first. For deep analysis, follow the documentation loading guide below.

---

## 1. System Overview

Strix is an open-source AI-driven penetration testing CLI (PyPI: `strix-agent`, v1.0.2). It runs autonomous AI agents inside a Docker sandbox to find and validate security vulnerabilities in web applications, APIs, repositories, or IP targets. Agents use a full hacker toolkit (Caido proxy, Kali tools, browser automation, Python runtime) and produce structured vulnerability reports with CVSS scores and working PoCs.

→ Deep detail: [docs/architecture/SYSTEM_OVERVIEW.md](docs/architecture/SYSTEM_OVERVIEW.md)

---

## 2. Repository Ownership

| Module | Owner Concern |
|--------|--------------|
| `strix/interface/` | CLI entry point, TUI, argument parsing, pre-flight checks |
| `strix/core/` | Scan runner, agent execution loop, coordinator, inputs, paths |
| `strix/agents/` | Agent factory, system prompt (Jinja2) renderer |
| `strix/tools/` | Host-side function tools registered with the SDK |
| `strix/config/` | Settings (pydantic-settings), file persistence |
| `strix/runtime/` | Docker sandbox lifecycle, Caido bootstrap |
| `strix/report/` | Scan artifact state, writer, deduplication |
| `strix/skills/` | Markdown skill files injected into agent prompts |
| `strix/telemetry/` | PostHog + Scarf usage telemetry |

→ Deep detail: [docs/architecture/MODULE_OWNERSHIP.md](docs/architecture/MODULE_OWNERSHIP.md)

---

## 3. High-Level Architecture

```
CLI/TUI  →  main.py (parse args, pre-flight)
         →  runner.run_strix_scan()
               │
               ├─ session_manager (Docker + Caido bootstrap)
               ├─ AgentCoordinator (graph state, snapshots)
               ├─ build_strix_agent() → SandboxAgent (openai-agents SDK)
               │     └─ system_prompt.jinja + skill Markdown files
               ├─ run_agent_loop() → Runner.run_streamed() → LLM via LiteLLM
               │     └─ tools: proxy, shell, filesystem, notes, todos,
               │               reporting, web_search, agents_graph, finish
               └─ ReportState (global artifact + callback store)
```

Agent graph: root agent spawns specialist child agents at runtime. Each agent runs in its own `asyncio.Task`, shares the same Docker container and `/workspace`, and communicates via the coordinator's message bus.

→ Deep detail: [docs/architecture/SERVICE_TOPOLOGY.md](docs/architecture/SERVICE_TOPOLOGY.md)

---

## 4. Entry Points

| Entry | Path | Purpose |
|-------|------|---------|
| CLI binary | `strix.interface.main:main` | Primary user entry point |
| Non-interactive (CI) | `run_cli()` in `strix/interface/cli.py` | Headless, exits with code 2 on findings |
| Interactive (TUI) | `run_tui()` in `strix/interface/tui/` | Textual-based interactive UI |
| Core scanner API | `run_strix_scan()` in `strix/core/runner.py` | Programmatic entry point |

---

## 5. Core Runtime Flow Summaries

**Fresh scan (non-interactive)**:
`main()` → parse args → Docker preflight → env validate → LLM warm-up → `run_strix_scan()` → provision container + Caido → build root `SandboxAgent` → `run_agent_loop()` → streaming SDK runner → tools (including `create_agent` to spawn children) → `finish_scan` tool exits root → `ReportState` writes artifacts to `strix_runs/<run-name>/`

**Resume scan**:
Same as above but `agents.json` + `agents.db` are loaded from prior run dir. Coordinator restored, child agents respawned, root session replayed via SDK history.

→ Deep detail: [docs/architecture/RUNTIME_FLOWS.md](docs/architecture/RUNTIME_FLOWS.md)

---

## 6. High-Risk Areas

| File | Risk | Why |
|------|------|-----|
| `strix/core/execution.py` | HIGH | Agent loop, resume logic, error recovery, image-strip retry |
| `strix/core/agents.py` | HIGH | `AgentCoordinator` global state, snapshot atomicity |
| `strix/report/state.py` | HIGH | Global `ReportState` singleton, artifact write ordering |
| `strix/agents/prompts/system_prompt.jinja` | HIGH | Controls all agent behavior |
| `strix/report/dedupe.py` | MEDIUM | Inline LLM call during vulnerability filing |
| `strix/runtime/session_manager.py` | MEDIUM | Container lifecycle, in-process cache |
| `strix/config/loader.py` | MEDIUM | Settings memoization, env/file precedence |
| `strix/tools/reporting/tool.py` | MEDIUM | Vulnerability schema validation and persistence |

→ Deep detail: [docs/architecture/CHANGE_BLAST_RADIUS.md](docs/architecture/CHANGE_BLAST_RADIUS.md)

---

## 7. Documentation Map

```
DOC_INDEX.md          — compressed router (load first)
CLAUDE.md             — this file (top-level engineering map)
docs/architecture/    — system structure, flows, dependencies
docs/ai/              — LLM pipeline, prompts, agent workflows
docs/operations/      — runbook, observability, failure modes
docs/security/        — trust model, auth, secrets
docs/integrations/    — third-party APIs and dependencies
docs/development/     — local dev, testing, safe change zones
```

---

## 8. Documentation Loading Guide

**General understanding**:
`DOC_INDEX.md` → `CLAUDE.md` → `docs/architecture/SYSTEM_OVERVIEW.md`

**Architecture tracing**:
`CLAUDE.md` → `docs/architecture/REPOSITORY_MAP.md` → `docs/architecture/RUNTIME_FLOWS.md` → `docs/architecture/DEPENDENCY_GRAPH.md`

**Runtime debugging**:
`CLAUDE.md` → `docs/architecture/RUNTIME_FLOWS.md` → `docs/operations/OBSERVABILITY.md` → `docs/operations/FAILURE_MODES.md` → `docs/architecture/CALL_GRAPH.md`

**Schema or contract changes**:
`CLAUDE.md` → `docs/architecture/DATA_CONTRACTS.md` → `docs/architecture/CHANGE_BLAST_RADIUS.md`

**Integration changes**:
`CLAUDE.md` → `docs/integrations/EXTERNAL_INTEGRATIONS.md` → `docs/architecture/SERVICE_TOPOLOGY.md`

**Security review**:
`CLAUDE.md` → `docs/security/SECURITY_MODEL.md` → `docs/security/TRUST_BOUNDARIES.md`

**AI prompt/model changes**:
`CLAUDE.md` → `docs/ai/AI_SYSTEM_MAP.md` → `docs/ai/PROMPTS.md` → `docs/ai/LLM_ARCHITECTURE.md` → `docs/ai/MODEL_GUARDRAILS.md` → `docs/ai/AI_PIPELINE_MAP.md`

**Production incident response**:
`CLAUDE.md` → `docs/operations/RUNBOOK.md` → `docs/operations/FAILURE_MODES.md` → `docs/operations/OPERATIONAL_FAILURE_GRAPH.md`

---

## 9. Recommended Reading Order

1. `DOC_INDEX.md` — domain index
2. `CLAUDE.md` — this file
3. `docs/architecture/SYSTEM_OVERVIEW.md` — what it does
4. `docs/architecture/REPOSITORY_MAP.md` — where things live
5. `docs/architecture/RUNTIME_FLOWS.md` — how a scan runs
6. `docs/ai/AGENT_WORKFLOWS.md` — multi-agent system
7. `docs/ai/PROMPTS.md` — prompt and skills system
8. `docs/architecture/CHANGE_BLAST_RADIUS.md` — what breaks what

---

## 10. Rules for Safe Changes

1. **Before editing `execution.py`**: read `RUNTIME_FLOWS.md` and `CALL_GRAPH.md` fully. The loop handles resume, interactive/non-interactive branching, image-strip retry, and lifecycle tool detection.
2. **Before editing the system prompt Jinja template**: read `PROMPTS.md` and `MODEL_GUARDRAILS.md`. Scope injection is security-critical.
3. **Before adding a tool**: read `MODULE_OWNERSHIP.md`. Tools are registered in `factory.py`; their errors must return as strings (model-visible), not raise.
4. **Before changing `ReportState`**: check `DATA_LINEAGE.md`. It is a global singleton and write-ordering matters.
5. **Before changing the `create_vulnerability_report` tool schema**: read `DATA_CONTRACTS.md`. The schema is enforced by both the SDK and the reporting tool; breaking it silently drops vulnerability reports.
6. **Before changing `session_manager.py`**: container teardown is best-effort; stranded containers are real risk.
7. **Do not change settings memoization** in `loader.py` without understanding that `_cached` is a module-level singleton reset only by `apply_config_override`.

→ Deep detail: [docs/development/SAFE_CHANGE_ZONES.md](docs/development/SAFE_CHANGE_ZONES.md)

---

## 11. Known Unknowns

- `strix/interface/tui/` not fully analyzed — TUI app (`tui/app.py`) event handling and renderer details may have additional state.
- Benchmarks in `benchmarks/` were not inspected — test coverage is unknown.
- `strix/tools/proxy/caido_api.py` wraps Caido SDK calls — the full proxy API surface was not analyzed.
- `strix/config/models.py` was not fully read — `configure_sdk_model_defaults` and `uses_chat_completions_tool_schema` logic unclear (model-specific tool schema branching).
- Skills Markdown content (under `strix/skills/`) was not fully read — the skills injected into agents drive much of the actual testing behavior.
- No formal test suite found — test coverage is unknown, possibly absent or in a separate repo.
- Resume behavior under partial-write scenarios (OS crash mid-snapshot) is not fully documented.
