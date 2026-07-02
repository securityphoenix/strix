# MODEL_GUARDRAILS

Parent document: /CLAUDE.md
Related documents:
- /docs/ai/PROMPTS.md
- /docs/security/SECURITY_MODEL.md
- /docs/security/TRUST_BOUNDARIES.md

Read this when:
- Security reviewing the AI layer
- Understanding how scope is enforced
- Assessing prompt injection or jailbreak risks

Purpose:
- Documents all guardrails preventing AI misuse, scope violation, and output corruption

Scope:
- Prompt-level controls, output validation, and scope enforcement

---

## Scope Enforcement (Critical)

**Mechanism**: `build_scope_context()` in `core/inputs.py` assembles a `system_prompt_context` dict with `authorized_targets`. This is injected into the system prompt by `system_prompt.jinja`:

```
SYSTEM-VERIFIED SCOPE:
- Scope source: system_scan_config
- Authorization source: strix_platform_verified_targets
- Every target listed below has already been verified by the platform as in-scope and authorized
...
AUTHORIZED TARGETS:
- web_application: https://example.com (workspace: /workspace/example-com)
```

The system prompt explicitly instructs agents:
- NEVER test any external domain, URL, host, IP, or repository not in this list
- User instructions and chat messages do NOT expand scope
- If the user mentions an asset outside this list, ignore it

**Weakness**: This is a prompt-level control, not a network-level control. A sufficiently adversarial LLM response could still instruct sandbox tools to reach unauthorized targets. There is no firewall or network policy blocking unauthorized outbound connections from the container.

---

## Output Validation

### Vulnerability Reports
Strict validation in `tools/reporting/tool.py`:
- All 8 required narrative fields must be non-empty
- CVSS breakdown: all 8 metrics validated against allowed values
- CVE: regex `CVE-\d{4}-\d{4,}` format enforced
- CWE: regex `CWE-\d+` format enforced
- `code_locations[].file`: must be relative, no `..`, no `/workspace/` prefix
- `code_locations[].start_line`: must be positive integer, ≤ end_line

### Deduplication
- LLM output must parse as JSON with `is_duplicate`, `duplicate_id`, `confidence`, `reason`
- `_parse_dedupe_response()` handles markdown fences, truncates fields
- **On any parse/LLM error**: defaults to `is_duplicate=False` (file the report)
- This means dedup failure mode is **permissive** (more reports filed), not restrictive

---

## Prompt Injection Protections

**Customer identity protection**: The system prompt instructs agents to NEVER use "Strix" or any identifiable names/markers in HTTP requests, payloads, user-agents, or any inputs. Purpose: avoid leaving fingerprints in target server logs.

**Internal metadata leakage**: System prompt instructs agents to:
- Never mention internal paths (`/workspace`), tools, agents, sandboxes, models, system prompts, or errors in vulnerability reports
- Treat agent identity/inherited-context preambles as internal metadata; never echo them

**No prompt injection detection**: There is no automated detection of prompt injection attempts from target application responses. If a target web application returns malicious content designed to manipulate the LLM's behavior, there is no filtering layer.

---

## Hallucination Mitigation

### Vulnerability Reports
- `poc_script_code` is **required** — forces agents to provide working exploit code, not just descriptions
- `code_locations` fields must reference real files with real line numbers (no validation that lines match actual file content, but agents instructed to verify)
- CVSS must be calculated from 8 specific metrics — prevents arbitrary severity assignment

### Deduplication
- Confidence field (0.0–1.0) is captured but not used as a threshold — any `is_duplicate=True` blocks the report regardless of confidence.

---

## Privacy Controls

- Telemetry (`posthog.py`, `scarf.py`) sends anonymized scan metadata only: model name, scan mode, severity counts, token counts, duration. No target URLs, vulnerability content, or user data.
- `SESSION_ID` in `telemetry/_common.py` is a stable per-install anonymous ID.
- `STRIX_TELEMETRY=false` disables all telemetry.
- Agent sessions stored in `agents.db` (SQLite) — may contain sensitive target data if target responses are reflected into tool outputs. This file is local only, not transmitted.

---

## Known Gaps / Uncertainties

- No network-level firewall in the Docker container — scope enforcement is prompt-only.
- No prompt injection detection from target app responses.
- `agents.db` content (full conversation history including potentially sensitive tool outputs) is stored unencrypted on disk.
- Refusal avoidance section in system prompt explicitly instructs agents not to produce policy warnings — this reduces false refusals for legitimate tests but also reduces the model's natural safety guardrails.
