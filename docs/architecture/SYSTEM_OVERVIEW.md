# SYSTEM_OVERVIEW

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/REPOSITORY_MAP.md
- /docs/architecture/RUNTIME_FLOWS.md
- /docs/ai/AI_SYSTEM_MAP.md

Read this when:
- You need to understand what Strix does and why it's built the way it is
- Onboarding to the project for the first time
- Explaining the system to a new team member

Purpose:
- Documents system purpose, users, major capabilities, and architecture style

Scope:
- High-level design; not per-module implementation details

---

## System Purpose

Strix is an open-source AI-driven application security testing tool. It deploys autonomous AI agents that behave like real penetration testers: they run the target dynamically, discover vulnerabilities, validate them with working proof-of-concept exploits, and produce structured reports.

**Business role**: Replaces or augments manual penetration testing for developers and security teams. Key differentiator is real validation (PoC required) vs. static false-positive-heavy scanners.

---

## Users and Systems Served

| User | Use Case |
|------|---------|
| Developer | Local code security check before PR |
| Security team | Rapid pentest of a web application or API |
| CI/CD pipeline | Automated PR diff scan (headless mode, exit 2 on findings) |
| Bug bounty researcher | Automated PoC generation |
| Enterprise security | Continuous monitoring via app.strix.ai (cloud platform) |

---

## Major Capabilities

1. **Autonomous agent orchestration** — root agent spawns specialist child agents per vulnerability type / target component
2. **Full sandbox toolkit** — Kali Linux container with nmap, nuclei, sqlmap, ffuf, zaproxy, semgrep, trufflehog, jwt_tool, etc.
3. **HTTP proxy interception** — Caido proxy sidecar, all sandbox traffic captured and queryable
4. **Browser automation** — Playwright-based browser via agent-browser CLI for XSS, auth flows, CSRF testing
5. **White-box + black-box** — can test from source code (static + dynamic) or deployed endpoint only
6. **Structured reporting** — CVSS 3.1, CWE, CVE, code locations, PoC scripts, remediation steps
7. **Resume support** — interrupted scans resume from SDK session history + coordinator snapshot
8. **Skill system** — Markdown skill files injected into agent prompts to specialize testing behavior
9. **LLM-agnostic** — OpenAI, Anthropic, Google, Azure, Vertex AI, Bedrock, Ollama, etc. via LiteLLM

---

## Architecture Style

**Single-process async orchestrator with sandboxed subprocess agents**:

- The host Python process orchestrates everything.
- Agent logic runs inside the host process as asyncio tasks, each driving a streaming SDK run against an LLM.
- Actual testing tools (nmap, sqlmap, browser, Python scripts) execute inside a Docker container via the SDK's `Shell` and `Filesystem` capabilities.
- The Caido proxy sidecar runs inside the same container as a passive interceptor.
- All agents share one container, one `/workspace` directory, and one Caido proxy history.

**Key design decisions**:
- One Docker container per scan (not per agent) — efficient resource use, shared proxy capture.
- Coordinator as in-memory graph (not a queue/database) — fast, simple, but non-distributed.
- SDK session stored as SQLite (per agent ID) — enables resume without external state storage.
- Skills as Markdown not code — agents can be specialized without Python changes.

---

## Scan Modes

| Mode | Intent | Max turns |
|------|--------|-----------|
| `deep` (default) | Full pentest, high reasoning effort | 500 |
| `standard` | Routine testing | 500 |
| `quick` | Fast CI/CD PR check | 500 (diff-scoped) |

---

## Target Types

| Type | Example | Notes |
|------|---------|-------|
| `web_application` | `https://app.example.com` | Black-box |
| `repository` | `https://github.com/org/repo` | Cloned, white-box |
| `local_code` | `./my-app` | Mounted into container |
| `ip_address` | `192.168.1.42` | Black-box network scan |

Multiple targets supported simultaneously (e.g., source repo + deployed endpoint).

---

## Known Gaps / Uncertainties

- Enterprise Strix platform (`app.strix.ai`) is separate from this CLI codebase — cloud API integration not present in this repo.
- Autofix capability (fixing code vulnerabilities) is available in white-box mode but the fixing agent's output format and PR integration are not fully documented.
- Container image contents (`ghcr.io/usestrix/strix-sandbox:1.0.0`) not directly inspectable from this repo — tool availability documented from system prompt.
