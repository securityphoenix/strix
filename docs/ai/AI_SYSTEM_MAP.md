# AI_SYSTEM_MAP

Parent document: /CLAUDE.md
Related documents:
- /docs/ai/PROMPTS.md
- /docs/ai/LLM_ARCHITECTURE.md
- /docs/ai/AGENT_WORKFLOWS.md
- /docs/ai/MODEL_GUARDRAILS.md

Read this when:
- Getting an overview of where and why AI is used in the system
- Assessing downstream dependencies on AI outputs

Purpose:
- Maps every AI/LLM touchpoint, why it exists, and what depends on its output

Scope:
- AI usage in the main scan pipeline and reporting; not the cloud platform

---

## AI Touchpoints

| Touchpoint | File | Model | Purpose | Output Consumed By |
|------------|------|-------|---------|-------------------|
| Root agent | `core/runner.py` via openai-agents SDK | `STRIX_LLM` | Orchestrates scan, spawns subagents | scan artifacts, child agents |
| Child agents (any count) | `core/execution.py` | `STRIX_LLM` | Specialist security testing | vulnerability reports, parent completion reports |
| LLM warm-up | `interface/main.py:warm_up_llm()` | `STRIX_LLM` | Connectivity test before scan | go/no-go for scan |
| Deduplication | `report/dedupe.py:check_duplicate()` | `STRIX_LLM` | Decides if a vulnerability is a duplicate | `create_vulnerability_report` acceptance/rejection |

---

## Boundary Between Deterministic and AI-Driven Behavior

| Behavior | Deterministic | AI-Driven |
|----------|--------------|-----------|
| Target type inference | ✓ | |
| Diff scope computation | ✓ | |
| CVSS score calculation | ✓ (cvss3 lib) | |
| CVE/CWE format validation | ✓ | |
| Vulnerability deduplication | | ✓ |
| Attack strategy and tool selection | | ✓ |
| Payload generation and mutation | | ✓ |
| Vulnerability root cause analysis | | ✓ |
| PoC exploit code | | ✓ |
| Report text (all narrative fields) | | ✓ |
| Agent spawn decisions (which specialists) | | ✓ |
| Scan completion decision (finish_scan) | | ✓ |

---

## Downstream Dependencies on AI Outputs

| AI Output | Downstream Consumer | Failure Mode |
|-----------|-------------------|--------------|
| `create_vulnerability_report` call | `report/state.py`, disk artifacts, UI callback | Vuln not filed; no error visible to user |
| `finish_scan` call | Run exits; `scan_completed` flag set | Scan never terminates; hits MaxTurnsExceeded |
| `agent_finish` call | Parent's inbox; parent decides next steps | Parent waits indefinitely (timeout at 600s) |
| `create_agent` call | New asyncio task spawned | More/fewer specialists than intended |
| Dedupe decision `is_duplicate=True` | Vulnerability not filed | False duplicate → real vuln silently dropped |
| Dedupe decision `is_duplicate=False` | Vulnerability filed | True duplicate → duplicate vulns in report |

---

## Known Gaps / Uncertainties

- No formal evaluation of deduplication accuracy — false positive/negative rates are unknown.
- Agent behavior under different LLM providers (GPT vs Claude vs Gemini) may differ significantly despite the same system prompt.
- Tool call reliability varies by provider — some models call lifecycle tools inconsistently.
