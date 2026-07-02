# RUNTIME_DEPENDENCY_TREE

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/SERVICE_TOPOLOGY.md
- /docs/operations/FAILURE_MODES.md
- /docs/operations/OPERATIONAL_FAILURE_GRAPH.md

Read this when:
- Diagnosing boot failures
- Understanding what must be healthy before a scan starts
- Planning infrastructure changes

Purpose:
- Startup and runtime dependency ordering

Scope:
- What must be available before each stage; not code structure

---

## Startup Dependency Tree

```
strix CLI starts
    │
    ├─ [1] Docker daemon (required)
    │       └─ checked via shutil.which("docker") + docker.from_env()
    │          Fails: sys.exit(1) with "DOCKER NOT INSTALLED" error
    │
    ├─ [2] Container image (required)
    │       └─ ghcr.io/usestrix/strix-sandbox:1.0.0 (or STRIX_IMAGE)
    │          If not present locally: pulled from GHCR (public registry)
    │          Fails: sys.exit(1) with "FAILED TO PULL IMAGE" error
    │
    ├─ [3] STRIX_LLM env var (required)
    │       └─ Validated before warmup
    │          Fails: sys.exit(1) with missing vars error
    │
    ├─ [4] LLM Provider (required — warmed up before scan)
    │       └─ asyncio.wait_for(model.get_response("Reply with OK"), timeout=300s)
    │          Fails: sys.exit(1) with "LLM CONNECTION FAILED" error
    │          Note: failure here prevents any scan from starting
    │
    └─ [5] Settings loaded (required — memoized after first call)
            └─ ~/.strix/cli-config.json (optional, created on first run)
               Fails gracefully: uses env defaults only

───────────────────────────────────────────────────────
Scan starts (run_strix_scan called)
───────────────────────────────────────────────────────
    │
    ├─ [6] Docker container running (required per scan)
    │       └─ StrixDockerSandboxClient.create() + session.start()
    │          Fails: exception propagates to caller
    │
    ├─ [7] Caido sidecar ready (required per scan)
    │       └─ bootstrap_caido(): retry loop (10 attempts, backoff up to 8s)
    │          Waits for Caido GraphQL at :48080 to accept loginAsGuest
    │          Fails: RuntimeError("loginAsGuest failed after N attempts")
    │
    └─ [8] Local source mounts complete (required for white-box scans)
            └─ LocalDir(src=host_path) entries materialized by session.start()
               Fails if host path doesn't exist at session.start() time
```

---

## Per-Scan Runtime Dependencies

| Component | Needed By | Failure Impact |
|-----------|-----------|----------------|
| Docker daemon | session_manager, StrixDockerSandboxClient | Scan cannot start |
| Container image | session_manager | Scan cannot start (pull attempted) |
| Caido sidecar | proxy tools (list_requests, repeat_request, etc.) | Proxy tools fail; scan may proceed without proxy history |
| LLM provider | Every agent turn | Agent turn fails; depends on SDK retry behavior |
| Perplexity API | web_search tool | web_search returns error; scan continues |
| PostHog/Scarf | Telemetry events | Events silently dropped; scan continues |
| `/workspace` | All agent filesystem operations | Agent cannot read/write test artifacts |

---

## Resume-Specific Dependencies

| Component | Required | Notes |
|-----------|---------|-------|
| `strix_runs/<run>/run.json` | YES — hard fail if missing | Contains targets, scan_mode, instruction |
| `strix_runs/<run>/.strix_state/agents.json` | YES — hard fail if missing (validated before run) | Agent graph snapshot |
| `strix_runs/<run>/.strix_state/agents.db` | YES — hard fail if missing | SDK conversation history |
| `strix_runs/<run>/vulnerabilities.json` | No — hydrated if present | Prior vulnerability list for dedup |
| Cloned repo at `cloned_repo_path` | YES if target was a repository | Validated before container start |
| `STRIX_IMAGE` from settings | YES | May differ from original run if settings changed |

---

## Known Gaps / Uncertainties

- Whether the Docker daemon must be the same version as used during original scan (for resume) is unknown.
- Network connectivity requirements for container → target during resume are assumed to be the same as fresh scan.
- What happens if the LLM provider changes between a fresh scan and its resume is undocumented.
