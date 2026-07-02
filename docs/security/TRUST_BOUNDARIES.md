# TRUST_BOUNDARIES

Parent document: /CLAUDE.md
Related documents:
- /docs/security/SECURITY_MODEL.md
- /docs/ai/MODEL_GUARDRAILS.md

Read this when:
- Threat modeling the system
- Reviewing trust relationships between components
- Assessing what data crosses what boundaries

Purpose:
- Internal vs external trust zones, privileged modules, sensitive ingress/egress

Scope:
- Runtime trust model; not organizational trust

---

## Trust Zones

```
Zone 1: TRUSTED — Host machine (user's OS process)
    - strix CLI Python process
    - ~/.strix/cli-config.json (0o600)
    - strix_runs/ output directory
    - Docker daemon socket (local only)

Zone 2: SEMI-TRUSTED — Docker sandbox container
    - Kali Linux with security tools
    - Agent-executed code (Python scripts, shell commands)
    - /workspace (mounted from host; bidirectional)
    - Caido proxy history (in-container only)
    NOTE: Container is controlled by LLM output, so its trust
    level equals the LLM's trustworthiness

Zone 3: UNTRUSTED — External systems
    - Test targets (web apps, APIs, repos)
    - LLM provider API responses
    - Target HTTP responses (may contain prompt injection)
    - Downloaded wordlists, tools from internet

Zone 4: CONDITIONALLY-TRUSTED — LLM Provider
    - Trusted for correctness of model responses
    - NOT trusted for security enforcement
    - Can be manipulated by adversarial target app responses
```

---

## Ingress Points

| Ingress | From | To | Risk |
|---------|------|-----|------|
| `--target` CLI arg | User | scan_config | LOW — type-checked, sanitized |
| `--instruction` CLI arg | User | LLM system context | MEDIUM — credentials may be present |
| `--instruction-file` | User | LLM system context | MEDIUM — file read, same risk as instruction |
| LLM provider responses | External API | Host-side tool execution | HIGH — drives all agent actions |
| Target app HTTP responses | Target | Container + LLM context | HIGH — potential prompt injection |
| Container exec output | Container | Host (tool results) | MEDIUM — tool errors caught as strings |
| `~/.strix/cli-config.json` | Disk | Host settings | LOW — 0o600; user-owned file |

---

## Egress Points

| Egress | From | To | Contains |
|--------|------|-----|---------|
| LLM API requests | Host | Provider (HTTPS) | System prompt, conversation history, tool schemas |
| Perplexity API requests | Host | Perplexity (HTTPS) | Search queries (may contain target info) |
| PostHog telemetry | Host | PostHog (HTTPS) | Anonymized scan metadata |
| Scarf telemetry | Host | Scarf (HTTPS) | Anonymized scan metadata |
| Container HTTP/S traffic | Container | Test targets (via Caido) | All scanning/testing traffic |
| Container tool downloads | Container | Internet | apt/pip/npm downloads |

---

## Privileged Modules

| Module | Privilege | Why |
|--------|-----------|-----|
| `runtime/docker_client.py` | Docker daemon access | Creates containers with NET_ADMIN/NET_RAW |
| `runtime/session_manager.py` | Docker daemon access | Container lifecycle |
| `interface/main.py:persist_current()` | Filesystem write | Writes API keys to disk |
| `core/execution.py` | asyncio task creation | Creates and cancels agent tasks |
| Any tool calling `exec_command` | Shell in container | Arbitrary code execution in container |

---

## Sensitive Data Crossing Boundaries

| Data | From Zone | To Zone | Notes |
|------|-----------|---------|-------|
| LLM API key | Host disk | LLM Provider | HTTPS; stored at rest in 0o600 file |
| Target credentials (via --instruction) | User → Host | LLM Provider | Stored in run.json; appears in LLM context |
| Target HTTP responses | Target | Container → LLM | May contain prompt injection payloads |
| Vulnerability PoC code | LLM | Host disk | Local only; not transmitted post-creation |
| Container command output | Container | Host (tool results) | May contain sensitive target data in LLM context |

---

## Implicit Trust Assumptions

1. **User running Strix is authorized to test the target** — no technical enforcement.
2. **LLM provider is honest** — a compromised provider could manipulate agent behavior.
3. **Container image is uncompromised** — `ghcr.io/usestrix/strix-sandbox:1.0.0` is trusted as-is.
4. **`agents.db` is safe on disk** — no encryption; assumed user controls their own filesystem.
5. **LiteLLM correctly routes to the configured provider** — no request verification.

---

## Known Gaps / Uncertainties

- No mTLS or request signing between host and LLM provider (beyond HTTPS + API key).
- Container filesystem access controls not documented — any agent process inside the container can write to `/workspace`.
- Whether the container can reach the host's loopback (beyond `host.docker.internal`) depends on Docker network configuration.
