# SECURITY_MODEL

Parent document: /CLAUDE.md
Related documents:
- /docs/security/TRUST_BOUNDARIES.md
- /docs/ai/MODEL_GUARDRAILS.md

Read this when:
- Security reviewing the codebase
- Assessing authorization model and secrets handling
- Understanding what the tool does and doesn't protect

Purpose:
- Auth, authz, secrets handling, trust assumptions, sensitive flows

Scope:
- CLI tool security model; not cloud platform

---

## Authorization Model

**Strix itself has no authentication or authorization layer.** It is a CLI tool that runs with the privileges of the executing user.

The tool assumes:
- The user running Strix has permission to test the specified targets (by convention and legal obligation, not by technical enforcement)
- The system prompt enforces scope boundaries within the LLM's behavior (prompt-level control only)

**No access control between features** — if you can run `strix`, you can test any target (modulo what the LLM will actually execute).

---

## Secrets Handling

| Secret | Storage | Transmission |
|--------|---------|-------------|
| `LLM_API_KEY` | Env var + `~/.strix/cli-config.json` (0o600) | Sent to LLM provider via HTTPS per-turn |
| `PERPLEXITY_API_KEY` | Env var + `~/.strix/cli-config.json` (0o600) | Sent to Perplexity API via HTTPS |
| Target credentials (via `--instruction`) | Passed to LLM system prompt | Sent to LLM; may appear in `agents.db` conversation history |
| Caido guest token | In-memory only (session lifetime) | HTTP to localhost only |

**Risk**: API keys are stored in `~/.strix/cli-config.json` with 0o600 permissions. Target credentials passed via `--instruction` are stored in `agents.db` (SQLite, unencrypted) and in `run.json` (`instruction` field).

---

## Sensitive Flows

| Flow | Sensitivity | Risk |
|------|------------|------|
| `--instruction` with credentials | HIGH | Credentials stored in run.json and agents.db, unencrypted on disk |
| LLM API key transmission | HIGH | HTTPS; key stored in 0o600 file |
| Vulnerability PoC code | MEDIUM | Stored in vulnerabilities.json; local only |
| Caido proxy capture | MEDIUM | All target HTTP traffic captured in Caido (lost on container teardown) |
| Telemetry | LOW | Anonymized; no PII, no target data |

---

## Docker Security

The sandbox container runs with **elevated capabilities**:
- `NET_ADMIN` — network administration (required for Caido traffic interception)
- `NET_RAW` — raw socket access (required for nmap/network scanning tools)
- `host.docker.internal` host gateway — container can reach host machine services

**Implication**: A compromised or malicious LLM response could, in theory, use `NET_ADMIN` and `NET_RAW` capabilities to perform network attacks from the container. The scope enforcement is prompt-level only.

---

## Supply Chain

| Component | Risk |
|-----------|------|
| `openai-agents[litellm]` | Core SDK; compromise would control all LLM interaction |
| `docker` (python SDK) | Container lifecycle; compromise could escape to host |
| `caido-sdk-client` | GraphQL client; compromise could exfiltrate proxy traffic |
| `ghcr.io/usestrix/strix-sandbox:1.0.0` | Container image; compromise would run malicious tools |

---

## No Security Controls Present

- No input sanitization of `--instruction` (passed directly to LLM)
- No output sanitization of LLM responses
- No prompt injection detection
- No container escape prevention beyond Docker's default isolation
- No network-level scope enforcement (firewall)
- No audit logging of what commands were run in the container

---

## Known Gaps / Uncertainties

- Whether `agents.db` (SQLite) is encrypted at rest: NO — plaintext SQLite file.
- Whether target credentials in `--instruction` are purged from disk after scan: NO — they remain in `run.json`.
- Container image provenance and signing not analyzed.
