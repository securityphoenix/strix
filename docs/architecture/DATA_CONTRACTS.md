# DATA_CONTRACTS

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/DATA_LINEAGE.md
- /docs/architecture/CHANGE_BLAST_RADIUS.md
- /docs/ai/PROMPTS.md

Read this when:
- Changing the vulnerability report schema
- Adding fields to run.json or vulnerabilities.json
- Modifying tool input/output contracts
- Changing settings keys or environment variable names

Purpose:
- Documents critical schemas, DTOs, tool contracts, and backward-compatibility concerns

Scope:
- Schemas derived from source code; external API schemas not covered

---

## 1. Settings / Configuration

**Source**: `strix/config/settings.py`
**Persistence**: `~/.strix/cli-config.json` as `{"env": {"KEY": "value"}}`

| Env Var | Type | Default | Required |
|---------|------|---------|----------|
| `STRIX_LLM` | string | None | **YES** |
| `LLM_API_KEY` | string | None | No (some providers) |
| `LLM_API_BASE` | string | None | No |
| `STRIX_REASONING_EFFORT` | `none\|minimal\|low\|medium\|high\|xhigh` | `high` | No |
| `LLM_TIMEOUT` | int (seconds) | `300` | No |
| `STRIX_IMAGE` | string | `ghcr.io/usestrix/strix-sandbox:1.0.0` | No |
| `STRIX_RUNTIME_BACKEND` | string | `docker` | No |
| `STRIX_TELEMETRY` | bool | `true` | No |
| `PERPLEXITY_API_KEY` | string | None | No |

Also accepts: `OPENAI_API_KEY`, `OPENAI_API_BASE`, `OPENAI_BASE_URL`, `LITELLM_BASE_URL`, `OLLAMA_API_BASE` as aliases.

**Backward compatibility concern**: Adding a new required env var breaks all existing users. New settings MUST have a default.

---

## 2. Run Record (run.json)

**Written by**: `report/writer.py:write_run_record()`
**Read by**: `interface/main.py:_load_resume_state()`, `report/state.py:hydrate_from_run_dir()`

```json
{
  "run_id": "web-application-abc123",
  "run_name": "web-application-abc123",
  "status": "running|completed|stopped|interrupted|failed",
  "start_time": "2024-01-01T12:00:00+00:00",
  "end_time": "2024-01-01T13:00:00+00:00",
  "targets_info": [
    {
      "type": "web_application|repository|local_code|ip_address",
      "details": {
        "target_url": "https://example.com",
        "workspace_subdir": "example-com",
        "cloned_repo_path": "/tmp/..."
      },
      "original": "https://example.com"
    }
  ],
  "instruction": "Focus on IDOR and auth",
  "scan_mode": "quick|standard|deep",
  "non_interactive": false,
  "local_sources": [{"source_path": "/local/path", "workspace_subdir": "app"}],
  "diff_scope": {"active": false, "repos": []},
  "scope_mode": "auto|diff|full",
  "diff_base": "origin/main",
  "scan_results": {
    "scan_completed": true,
    "executive_summary": "...",
    "methodology": "...",
    "technical_analysis": "...",
    "recommendations": "...",
    "success": true
  },
  "llm_usage": {
    "requests": 142,
    "input_tokens": 500000,
    "output_tokens": 50000,
    "total_tokens": 550000,
    "cost": 1.25
  }
}
```

**Backward compatibility concern**: `_load_resume_state()` reads `targets_info` — changing this field breaks resume. `hydrate_from_run_dir()` reads `start_time`, `end_time`, `scan_results`, `llm_usage` — these must stay stable.

---

## 3. Vulnerability Report (vulnerabilities.json)

**Written by**: `report/writer.py:write_vulnerabilities()`
**Read by**: `report/state.py:hydrate_from_run_dir()` for resume dedup

```json
[
  {
    "id": "vuln-0001",
    "title": "SQL Injection in /api/login",
    "severity": "critical|high|medium|low|info",
    "timestamp": "2024-01-01 12:30:00 UTC",
    "description": "...",
    "impact": "...",
    "target": "https://example.com",
    "technical_analysis": "...",
    "poc_description": "Step-by-step reproduction...",
    "poc_script_code": "import requests\n...",
    "remediation_steps": "...",
    "cvss": 9.8,
    "cvss_breakdown": {
      "attack_vector": "N",
      "attack_complexity": "L",
      "privileges_required": "N",
      "user_interaction": "N",
      "scope": "U",
      "confidentiality": "H",
      "integrity": "H",
      "availability": "H"
    },
    "endpoint": "/api/login",
    "method": "POST",
    "cve": "CVE-2024-1234",
    "cwe": "CWE-89",
    "code_locations": [
      {
        "file": "src/auth/login.py",
        "start_line": 42,
        "end_line": 45,
        "snippet": "query = f\"SELECT * FROM users WHERE username='{username}'\"",
        "label": "Vulnerable query construction",
        "fix_before": "...",
        "fix_after": "..."
      }
    ],
    "agent_id": "a1b2c3d4",
    "agent_name": "SQLi Validation Agent"
  }
]
```

**Backward compatibility concern**: The `id` field (`vuln-NNNN`) is used for deduplication lookup across resume. Changing allocation logic would cause duplicate IDs.

**Critical invariant**: `code_locations[].file` must be relative path (no leading `/`, no `..`, no `/workspace/` prefix). The tool validates this strictly.

---

## 4. create_vulnerability_report Tool Input

**Enforced by**: `tools/reporting/tool.py`
**All fields required** (empty string rejected):

| Field | Type | Notes |
|-------|------|-------|
| `title` | string | No CVE number in title |
| `description` | string | Discovery + what it is |
| `impact` | string | Business risk |
| `target` | string | Affected URL/domain/repo |
| `technical_analysis` | string | Root cause |
| `poc_description` | string | Step-by-step reproduction |
| `poc_script_code` | string | Working exploit (REQUIRED) |
| `remediation_steps` | string | Specific, actionable |
| `cvss_breakdown` | dict | 8 CVSS 3.1 metrics |
| `endpoint` | string? | API path |
| `method` | string? | HTTP method |
| `cve` | string? | `CVE-YYYY-NNNNN` format |
| `cwe` | string? | `CWE-NNN` format |
| `code_locations` | list? | White-box only |

---

## 5. Agent Coordinator Snapshot (agents.json)

**Written by**: `core/agents.py:AgentCoordinator._maybe_snapshot()` (atomic via temp file + rename)
**Read by**: `core/runner.py:run_strix_scan()` for resume

```json
{
  "statuses": {"a1b2c3d4": "running|waiting|completed|stopped|crashed|failed"},
  "parent_of": {"a1b2c3d4": null, "b2c3d4e5": "a1b2c3d4"},
  "names": {"a1b2c3d4": "strix", "b2c3d4e5": "SQLi Agent"},
  "metadata": {
    "a1b2c3d4": {"task": "...", "skills": []},
    "b2c3d4e5": {"task": "Test SQLi in /api/login", "skills": ["sql_injection"]}
  },
  "pending_counts": {"a1b2c3d4": 0, "b2c3d4e5": 1}
}
```

**Backward compatibility concern**: Resume depends on this schema. Adding new fields is safe. Removing or renaming existing fields breaks resume.

---

## 6. SDK Session Database (agents.db)

**Created by**: `core/sessions.py:open_agent_session()` via `agents.SQLiteSession`
**Format**: SQLite, schema owned by `openai-agents` SDK
**Purpose**: Stores full LLM conversation history (input messages + model responses + tool calls) per agent ID

**Backward compatibility concern**: Upgrading `openai-agents` SDK may change the SQLite schema, breaking resume from prior runs built against the old schema. No migration path documented.

---

## Known Gaps / Uncertainties

- `strix/config/models.py` schema (model normalization, chat completions tool schema branching) not fully documented.
- `executive_report.md` format (written by `write_executive_report`) not analyzed.
- `tools/proxy/tools.py` output schemas (what `list_requests`, `view_request` return to the LLM) not documented.
