# EXTERNAL_INTEGRATIONS

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/SERVICE_TOPOLOGY.md
- /docs/security/TRUST_BOUNDARIES.md

Read this when:
- Changing a third-party dependency
- Debugging a connectivity issue with an external service
- Assessing dependency risk

Purpose:
- Documents all third-party APIs, services, and libraries — what each is used for and failure/contract risks

Scope:
- Runtime integrations; not dev-tool dependencies (ruff, mypy, etc.)

---

## LLM Providers (via openai-agents + LiteLLM)

**Configured via**: `STRIX_LLM`, `LLM_API_KEY`, `LLM_API_BASE`
**Used by**: Every agent turn (streaming model calls)
**Contract**: OpenAI-compatible chat/completions API (via LiteLLM abstraction)

| Provider | STRIX_LLM example | Auth |
|---------|-------------------|------|
| OpenAI | `openai/gpt-5.4` | `LLM_API_KEY` |
| Anthropic | `anthropic/claude-sonnet-4-6` | `LLM_API_KEY` |
| Google Vertex AI | `vertex_ai/gemini-3-pro-preview` | ADC / service account |
| AWS Bedrock | `bedrock/...` | AWS credentials |
| Azure OpenAI | `azure/...` | `LLM_API_KEY` + `LLM_API_BASE` |
| Ollama | `ollama/...` | `LLM_API_BASE=http://localhost:11434` |

**Failure risk**: Provider outage → warm-up fails → scan cannot start. Mid-scan failure → agent crashes or fails.
**Contract risk**: LiteLLM API changes between `openai-agents` SDK versions could break model routing.

---

## Perplexity API

**Configured via**: `PERPLEXITY_API_KEY`
**Used by**: `strix/tools/web_search/tool.py`
**Purpose**: Real-time web search for payload research, vulnerability techniques, CVE lookup
**Failure mode**: `web_search` tool returns error string; scan continues without web context (degraded)

---

## Docker (python-docker SDK + Docker daemon)

**Package**: `docker>=7.1.0`
**Used by**: `runtime/backends.py`, `runtime/docker_client.py`, `runtime/session_manager.py`
**Purpose**: Container lifecycle — create, start, exec, delete
**Failure risk**: Docker SDK version mismatch with daemon → container operations fail → scan cannot start.
**Contract risk**: `DockerSandboxClientOptions`, `StrixDockerSandboxClient` depend on `agents.sandbox.sandboxes.docker` SDK internals.

---

## Caido (caido-sdk-client)

**Package**: `caido-sdk-client>=0.2.0`
**Used by**: `runtime/caido_bootstrap.py`, `tools/proxy/caido_api.py`, `tools/proxy/tools.py`
**Purpose**: HTTP proxy interception — capture all scan traffic, query history, repeat requests
**Protocol**: GraphQL over HTTP to container-exposed port
**Auth**: Guest token from `loginAsGuest` mutation
**Failure risk**: Caido bootstrap fails → scan cannot start (hard dependency). Mid-scan failure → proxy tools fail → degraded scan.
**Contract risk**: Caido GraphQL API changes → all proxy tools break.

---

## openai-agents SDK

**Package**: `openai-agents[litellm]==0.14.6` (pinned)
**Used by**: `core/runner.py`, `core/execution.py`, `core/sessions.py`, `agents/factory.py`
**Purpose**: Agent runner, session storage (SQLiteSession), streaming runs, sandbox capabilities
**Failure risk**: SDK version upgrade may change `agents.db` SQLite schema → breaks resume for prior runs.
**Contract risk**: `SandboxAgent`, `Runner`, `RunConfig`, `SandboxRunConfig`, `Filesystem`, `Shell` APIs are core dependencies. Any change in these APIs requires coordinated updates.

---

## cvss (CVSS 3.1 calculation)

**Package**: `cvss>=3.2`
**Used by**: `tools/reporting/tool.py:_calculate_cvss()`
**Purpose**: Calculate CVSS 3.1 score from 8-metric breakdown
**Failure mode**: Falls back to score=7.5/severity=high if library raises

---

## GitHub Container Registry (ghcr.io)

**URL**: `ghcr.io/usestrix/strix-sandbox:1.0.0`
**Used by**: `pull_docker_image()` in `interface/main.py`
**Purpose**: Source of the Kali Linux sandbox image
**Failure risk**: Registry outage → first-run pull fails → scan cannot start. Mitigated by local image cache after first pull.

---

## PostHog

**URL**: `https://us.i.posthog.com/capture/`
**Used by**: `telemetry/posthog.py`
**Purpose**: Usage analytics (scan counts, model usage, finding severity distribution)
**Failure mode**: Fire-and-forget; any error silently swallowed. Zero impact on scan.

---

## Scarf

**Used by**: `telemetry/scarf.py`
**Purpose**: PyPI install analytics
**Failure mode**: Fire-and-forget; zero impact on scan.

---

## Known Gaps / Uncertainties

- `tools/proxy/caido_api.py` full API surface not analyzed.
- Perplexity API endpoint and response format not confirmed from source (used via `web_search` tool).
- Whether `openai-agents` v0.14.6 SQLite schema is documented anywhere is unknown — assumes forward-compatible.
