# DATA_LINEAGE

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/DATA_CONTRACTS.md
- /docs/architecture/CHANGE_BLAST_RADIUS.md

Read this when:
- Tracing where a piece of data comes from and where it ends up
- Investigating data loss or corruption issues
- Understanding how vulnerability data flows from LLM to disk

Purpose:
- Where key data originates, how it is transformed, where it is stored, where it is consumed

Scope:
- Vulnerability reports, scan state, agent sessions, settings

---

## Vulnerability Data Lineage

```
Origin: LLM output (agent reasoning)
   │
   ▼
Agent calls create_vulnerability_report(...) tool
   │  Fields: title, description, impact, target, technical_analysis,
   │          poc_description, poc_script_code, remediation_steps,
   │          cvss_breakdown, endpoint, method, cve, cwe, code_locations
   ▼
tools/reporting/tool.py: validation + CVSS calculation
   │  Transformed: cvss_breakdown → cvss_score + severity via cvss3 lib
   │  Validated: required fields, CVSS metrics, CVE/CWE format, code_location paths
   ▼
report/dedupe.py: LLM-based deduplication check
   │  Consumed: existing vulnerability list from ReportState
   │  Produces: {is_duplicate, duplicate_id, confidence, reason}
   │  If duplicate: flow STOPS HERE, report not filed
   ▼
report/state.py: ReportState.add_vulnerability_report()
   │  Stored in-memory: self.vulnerability_reports list
   │  report_id assigned: f"vuln-{len(vulnerability_reports)+1:04d}"
   ▼
  ├─ Interface callback (live UI)
  │   vulnerability_found_callback(report) → CLI console print / TUI widget
  │
  └─ report/state.py: save_run_data()
       │
       ├─ report/writer.py: write_vulnerabilities()
       │     Writes: strix_runs/<run>/vulnerabilities.json  (full list)
       │     Writes: strix_runs/<run>/vuln-NNNN.md  (per-vuln Markdown)
       │     Tracks: _saved_vuln_ids to avoid overwriting on partial saves
       │
       └─ report/writer.py: write_run_record()
             Writes: strix_runs/<run>/run.json  (no vulnerability data, just metadata)
```

---

## Scan Config Lineage

```
Origin: CLI args (interface/main.py:parse_arguments())
   │
   ▼
scan_config dict assembled in run_cli() / run_tui()
   │  Fields: scan_id, targets, user_instructions, run_name, diff_scope,
   │           scan_mode, non_interactive, local_sources, scope_mode, diff_base
   ▼
report/state.py: ReportState.set_scan_config()
   │  Stored: run_record updated, status set to "running"
   ▼
report/writer.py: write_run_record()
   Writes: strix_runs/<run>/run.json

Also passed to:
  core/runner.py: run_strix_scan(scan_config=...)
    → core/inputs.py: build_root_task() → LLM initial input
    → core/inputs.py: build_scope_context() → system prompt injection
```

---

## LLM Usage Lineage

```
Origin: openai-agents SDK Usage objects (returned per model response)
   │
   ▼
core/hooks.py: ReportUsageHooks.on_tool_end() / on_agent_end()
   │  Calls: report_state.record_sdk_usage(agent_id, usage, model)
   ▼
report/state.py: LLMUsageLedger.record()
   │  Aggregated: per-agent and global token counts
   ▼
report/state.py: _sync_llm_usage_record() → run_record["llm_usage"]
   ▼
report/writer.py: write_run_record()
   Writes: strix_runs/<run>/run.json → llm_usage section

Also reported via:
  telemetry/posthog.py: scan_ended event (llm_requests, llm_tokens, llm_cost)
```

---

## Agent State Lineage

```
Origin: core/runner.py: coordinator.register() at scan start
   │
   ▼
core/agents.py: AgentCoordinator in-memory state
   │  statuses, parent_of, names, metadata, pending_counts
   ▼
core/agents.py: _maybe_snapshot() (called on every state change)
   │  Atomic write via temp file + rename
   Writes: strix_runs/<run>/.strix_state/agents.json

SDK conversation history:
  core/sessions.py: open_agent_session() → agents.SQLiteSession
  Writes: strix_runs/<run>/.strix_state/agents.db (per agent_id)
```

---

## Settings Data Lineage

```
Origin: Environment variables OR ~/.strix/cli-config.json
   │
   ▼
config/loader.py: load_settings()
   │  Precedence: env vars > JSON file > field defaults
   │  Memoized: _cached module-level variable
   ▼
Used by: all modules that call load_settings()

Persistence path (interface/main.py:persist_current()):
  Current env vars → written back to ~/.strix/cli-config.json
  Only non-empty env vars are persisted
  File permissions: 0o600
```

---

## Known Gaps / Uncertainties

- How `executive_report.md` (written by `write_executive_report`) is generated from `scan_results` not fully traced.
- The `notes` and `todos` in-memory + disk persistence (hydrated from state_dir) not fully analyzed.
- Telemetry data sent to PostHog/Scarf is not stored locally — fire-and-forget only.
