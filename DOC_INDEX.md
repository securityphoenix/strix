# DOC_INDEX — Strix Documentation Router

**Repo**: `strix-phx` (PyPI: `strix-agent` v1.0.2)
**Purpose**: Open-source AI penetration testing CLI — autonomous agents that find and validate security vulnerabilities.
**Stack**: Python 3.12+, openai-agents SDK, LiteLLM, Docker (Kali sandbox), Caido proxy, Textual TUI.

---

## Documentation Domains

| Domain | Authoritative Document | Load When |
|--------|----------------------|-----------|
| System purpose & capabilities | [SYSTEM_OVERVIEW.md](docs/architecture/SYSTEM_OVERVIEW.md) | First read |
| Module layout & ownership | [REPOSITORY_MAP.md](docs/architecture/REPOSITORY_MAP.md) | Navigating unfamiliar code |
| Runtime execution flows | [RUNTIME_FLOWS.md](docs/architecture/RUNTIME_FLOWS.md) | Tracing scan lifecycle |
| Module dependency graph | [DEPENDENCY_GRAPH.md](docs/architecture/DEPENDENCY_GRAPH.md) | Understanding coupling |
| Call graph from entry points | [CALL_GRAPH.md](docs/architecture/CALL_GRAPH.md) | Debugging runtime paths |
| Data schemas & contracts | [DATA_CONTRACTS.md](docs/architecture/DATA_CONTRACTS.md) | Schema or API changes |
| Service topology | [SERVICE_TOPOLOGY.md](docs/architecture/SERVICE_TOPOLOGY.md) | Infrastructure changes |
| Startup dependencies | [RUNTIME_DEPENDENCY_TREE.md](docs/architecture/RUNTIME_DEPENDENCY_TREE.md) | Boot order issues |
| Data lineage | [DATA_LINEAGE.md](docs/architecture/DATA_LINEAGE.md) | Data flow tracing |
| Change blast radius | [CHANGE_BLAST_RADIUS.md](docs/architecture/CHANGE_BLAST_RADIUS.md) | Risk assessment before editing |
| Module ownership | [MODULE_OWNERSHIP.md](docs/architecture/MODULE_OWNERSHIP.md) | Who owns what |
| AI usage map | [AI_SYSTEM_MAP.md](docs/ai/AI_SYSTEM_MAP.md) | AI touchpoints overview |
| Prompt inventory | [PROMPTS.md](docs/ai/PROMPTS.md) | Prompt changes |
| LLM request pipeline | [LLM_ARCHITECTURE.md](docs/ai/LLM_ARCHITECTURE.md) | Model/provider changes |
| Agent orchestration workflows | [AGENT_WORKFLOWS.md](docs/ai/AGENT_WORKFLOWS.md) | Multi-agent behavior |
| Prompt guardrails | [MODEL_GUARDRAILS.md](docs/ai/MODEL_GUARDRAILS.md) | Security review of AI layer |
| End-to-end AI pipeline | [AI_PIPELINE_MAP.md](docs/ai/AI_PIPELINE_MAP.md) | Full AI flow tracing |
| Operations runbook | [RUNBOOK.md](docs/operations/RUNBOOK.md) | Day-to-day ops |
| Observability (logging/telemetry) | [OBSERVABILITY.md](docs/operations/OBSERVABILITY.md) | Debugging, monitoring |
| Failure modes | [FAILURE_MODES.md](docs/operations/FAILURE_MODES.md) | Incident response |
| Deployment | [DEPLOYMENT.md](docs/operations/DEPLOYMENT.md) | Build & release |
| Failure propagation graph | [OPERATIONAL_FAILURE_GRAPH.md](docs/operations/OPERATIONAL_FAILURE_GRAPH.md) | Cascading failure analysis |
| Auth & secrets model | [SECURITY_MODEL.md](docs/security/SECURITY_MODEL.md) | Security review |
| Trust boundaries | [TRUST_BOUNDARIES.md](docs/security/TRUST_BOUNDARIES.md) | Threat modeling |
| External integrations | [EXTERNAL_INTEGRATIONS.md](docs/integrations/EXTERNAL_INTEGRATIONS.md) | Third-party dependency changes |
| Local dev setup | [LOCAL_DEVELOPMENT.md](docs/development/LOCAL_DEVELOPMENT.md) | Onboarding |
| Testing strategy | [TESTING_STRATEGY.md](docs/development/TESTING_STRATEGY.md) | Writing/running tests |
| Safe change zones | [SAFE_CHANGE_ZONES.md](docs/development/SAFE_CHANGE_ZONES.md) | Risk before editing |

---

## Task-Based Quick Routes

**General orientation** → `DOC_INDEX.md` → `CLAUDE.md` → `SYSTEM_OVERVIEW.md`

**Trace a scan** → `CLAUDE.md` → `RUNTIME_FLOWS.md` → `CALL_GRAPH.md`

**Add/change a tool** → `REPOSITORY_MAP.md` → `MODULE_OWNERSHIP.md` → `CHANGE_BLAST_RADIUS.md`

**Change the LLM/model** → `LLM_ARCHITECTURE.md` → `AI_SYSTEM_MAP.md` → `EXTERNAL_INTEGRATIONS.md`

**Edit system prompt / skills** → `PROMPTS.md` → `AGENT_WORKFLOWS.md` → `MODEL_GUARDRAILS.md`

**Debug a crash** → `FAILURE_MODES.md` → `OBSERVABILITY.md` → `RUNTIME_FLOWS.md`

**Security review** → `SECURITY_MODEL.md` → `TRUST_BOUNDARIES.md` → `MODEL_GUARDRAILS.md`

**Schema/contract change** → `DATA_CONTRACTS.md` → `CHANGE_BLAST_RADIUS.md` → `DATA_LINEAGE.md`

**Docker/sandbox change** → `SERVICE_TOPOLOGY.md` → `RUNTIME_DEPENDENCY_TREE.md` → `EXTERNAL_INTEGRATIONS.md`

**Onboarding** → `LOCAL_DEVELOPMENT.md` → `REPOSITORY_MAP.md` → `RUNTIME_FLOWS.md`

---

## High-Risk Areas (Always Read Deeper Docs Before Touching)

| Area | Risk | Deeper Doc |
|------|------|-----------|
| `strix/core/execution.py` | Agent loop, resume, error recovery | `RUNTIME_FLOWS.md`, `CHANGE_BLAST_RADIUS.md` |
| `strix/core/agents.py` | Coordinator state (global singleton) | `AGENT_WORKFLOWS.md` |
| `strix/report/state.py` | Global scan artifact state | `DATA_LINEAGE.md`, `DATA_CONTRACTS.md` |
| `strix/agents/prompts/system_prompt.jinja` | All agent behavior | `PROMPTS.md`, `MODEL_GUARDRAILS.md` |
| `strix/report/dedupe.py` | LLM dedupe call (inline, synchronous) | `AI_PIPELINE_MAP.md` |
| `strix/runtime/session_manager.py` | Container lifecycle | `OPERATIONAL_FAILURE_GRAPH.md` |
| `strix/config/loader.py` | Settings resolution & persistence | `DATA_CONTRACTS.md` |
| `strix/tools/reporting/tool.py` | Vulnerability filing contract | `DATA_CONTRACTS.md` |
