# OBSERVABILITY

Parent document: /CLAUDE.md
Related documents:
- /docs/operations/RUNBOOK.md
- /docs/operations/FAILURE_MODES.md

Read this when:
- Debugging scan behavior
- Monitoring token usage
- Understanding what telemetry is sent

Purpose:
- Logging, telemetry, and debugging hooks

Scope:
- Host-side observability; not sandbox container internals

---

## Logging

### Per-Scan Log File
- **Location**: `strix_runs/<run-name>/strix-<run-name>.log`
- **Created by**: `telemetry/logging.py:setup_scan_logging()`
- **Torn down by**: `teardown_logging()` in `run_strix_scan()` finally block
- **Contents**: All Python log events at DEBUG+ level for the scan duration, including:
  - Agent registration, status changes
  - Sandbox session creation/cleanup
  - Tool call results (DEBUG level)
  - LLM warm-up and model resolution
  - Vulnerability report creation
  - Coordinator snapshots

### Standard Python Logging
- Uses `logging.getLogger(__name__)` pattern throughout
- `configure_dependency_logging()` (called in `main()`) configures library loggers
- No file handler beyond the per-scan log; console output goes through Rich, not logging

---

## Telemetry

### PostHog (`strix/telemetry/posthog.py`)
**Endpoint**: `https://us.i.posthog.com/capture/`
**Events**:

| Event | When | Key Properties |
|-------|------|---------------|
| `scan_started` | Before scan | model, scan_mode, scan_type (whitebox/blackbox), interactive, has_instructions, first_run |
| `finding_reported` | Each `add_vulnerability_report()` | severity |
| `scan_ended` | On `finish_scan` or cleanup | exit_reason, duration_seconds, vulnerabilities_total/by_severity, llm_requests, llm_tokens, llm_cost |
| `error` | Unhandled exception | error_type, error_msg |

**Disable**: `STRIX_TELEMETRY=false`

### Scarf (`strix/telemetry/scarf.py`)
- Similar event pattern to PostHog (parallel fire-and-forget)
- Used for PyPI install analytics

### No Telemetry Contains
- Target URLs, IP addresses, or repository names
- Vulnerability titles, descriptions, or PoC code
- User instructions
- API keys or credentials

---

## Token Usage Tracking

**Mechanism**: `core/hooks.py:ReportUsageHooks` implements SDK `RunHooks` interface.
- `on_tool_end()` and `on_agent_end()` extract `Usage` from SDK events
- Calls `report_state.record_sdk_usage(agent_id, usage, model)`
- `LLMUsageLedger` in `report/usage.py` aggregates per-agent and global totals
- Written to `run.json:llm_usage` on every vulnerability filing or scan completion

**Reading usage during a scan**:
- Interactive TUI: displays live stats in status bar (via `interface/utils.py:build_live_stats_text()`)
- Post-scan: inspect `strix_runs/<run>/run.json` → `llm_usage` section

---

## Debugging Hooks

### Agent Graph Snapshot
`agents.json` is updated atomically on every coordinator state change. Inspect it to see the current agent tree and statuses.

### Live Agent Viewing (Interactive Mode)
TUI shows live agent graph and tool calls as they happen.

### Tool Call Debugging
Set Python logging level to DEBUG to see tool call inputs/outputs in the per-scan log.

### Caido Proxy History
All HTTP traffic from the sandbox is captured by Caido. During an interactive scan, agents can query proxy history via `list_requests`, `view_request`.
After scan ends, the Caido project is temporary and discarded with the container.

---

## Known Gaps / Uncertainties

- No alerting or metric export — telemetry is one-way to PostHog/Scarf.
- No Prometheus/OpenTelemetry integration.
- Caido proxy history is lost on container cleanup — not persisted to disk after scan.
- `configure_dependency_logging()` behavior (exact levels set) not confirmed.
