# FAILURE_MODES

Parent document: /CLAUDE.md
Related documents:
- /docs/operations/RUNBOOK.md
- /docs/operations/OPERATIONAL_FAILURE_GRAPH.md
- /docs/architecture/RUNTIME_DEPENDENCY_TREE.md

Read this when:
- Diagnosing a scan that didn't complete
- Investigating an unexpected error
- Planning incident response

Purpose:
- Common failure scenarios, symptoms, causes, and recovery steps

Scope:
- CLI/scan failures; not cloud platform

---

## Failure 1: "DOCKER NOT INSTALLED" or Docker daemon not running

**Symptom**: `sys.exit(1)` with "Docker not installed" or `DockerException` during pre-flight.
**Cause**: Docker CLI not in PATH, or Docker daemon not running.
**Recovery**:
```bash
# Check Docker
docker info
# Start Docker Desktop or daemon
# Then retry scan
```

---

## Failure 2: "FAILED TO PULL IMAGE"

**Symptom**: `sys.exit(1)` with "FAILED TO PULL IMAGE" panel.
**Cause**: No internet access, GHCR rate limit, or image `ghcr.io/usestrix/strix-sandbox:1.0.0` doesn't exist.
**Recovery**:
```bash
# Check connectivity
docker pull ghcr.io/usestrix/strix-sandbox:1.0.0
# Or use a cached image:
export STRIX_IMAGE="local-sandbox:latest"
```

---

## Failure 3: "LLM CONNECTION FAILED"

**Symptom**: `sys.exit(1)` with "LLM CONNECTION FAILED" panel during warm-up.
**Cause**: Invalid `LLM_API_KEY`, wrong `STRIX_LLM` format, provider outage, or timeout (300s default).
**Recovery**:
```bash
# Verify env vars
echo $STRIX_LLM
echo $LLM_API_KEY
# Test connectivity manually (provider-specific)
# Increase timeout if needed:
export LLM_TIMEOUT=600
```

---

## Failure 4: Scan hangs indefinitely

**Symptom**: No output, no new log entries, scan appears stuck.
**Cause**: Agent is in `waiting` state (non-interactive) waiting for a child that crashed; OR `wait_for_message` timeout not reached; OR LLM API stalled.
**Recovery**:
```bash
# Interrupt with Ctrl+C
# Check last log entries in strix_runs/<run>/strix-<run>.log
# Check agents.json for which agents are in which state
cat strix_runs/<run-name>/.strix_state/agents.json
# Resume if desired
strix --resume <run-name>
```

---

## Failure 5: "MaxTurnsExceeded" — scan ended without report

**Symptom**: Scan ends but no `executive_report.md` or `scan_completed=true` in `run.json`.
**Log indicator**: "Scan ended without calling finish_scan" in log file.
**Cause**: Agent produced text-only output instead of lifecycle tool call, exhausted recovery attempts.
**Recovery**:
```bash
# Resume to continue the scan
strix --resume <run-name>
# Or reduce max turns if testing:
# (no CLI flag; requires code change to DEFAULT_MAX_TURNS in core/inputs.py)
```

---

## Failure 6: Caido bootstrap fails

**Symptom**: `RuntimeError: loginAsGuest failed after 10 attempts`
**Cause**: Caido sidecar didn't start within retry window; container startup too slow; port conflict.
**Recovery**:
```bash
# Check container logs
docker logs $(docker ps -lq)
# The container must expose port 48080 and Caido must bind to it
# If recurring, file a bug — Caido readiness is environment-dependent
```

---

## Failure 7: Resume fails — agents.json missing

**Symptom**: `--resume <run>: missing .strix_state/agents.json`
**Cause**: Scan was interrupted before first agent snapshot was written (very early crash).
**Recovery**:
```bash
# Cannot resume; start fresh
strix --target <original-target>
# Or use the same run name if you want to reuse the run dir
```

---

## Failure 8: Resume fails — cloned repo missing

**Symptom**: `--resume <run>: cloned repo at /tmp/... is missing`
**Cause**: `/tmp` was cleaned between original scan and resume attempt.
**Recovery**:
```bash
# Re-clone manually and update run.json
# Or start a fresh scan
strix --target <repo-url>
```

---

## Failure 9: Vulnerability report "corrupt vulnerabilities.json"

**Symptom**: RuntimeError on scan startup: "vulnerabilities.json ... is corrupt"
**Cause**: Disk write was interrupted mid-write (OS crash, disk full) corrupting JSON.
**Recovery**:
```bash
# Inspect and manually repair the JSON
cat strix_runs/<run>/vulnerabilities.json | python3 -m json.tool
# Or delete it (lost findings) and resume
rm strix_runs/<run>/vulnerabilities.json
strix --resume <run>
```

---

## Failure 10: Tool errors returned to LLM

**Symptom**: Agent behavior seems confused; log shows "Tool X failed; returning error as result".
**Cause**: Expected — tool errors are intentionally returned as strings to the LLM rather than raising. LLM may recover or continue incorrectly.
**Recovery**: No action required for isolated tool errors. If systematic, check proxy connectivity, Docker exec access, or skill guidance.

---

## Failure 11: Container not cleaned up (stranded)

**Symptom**: `docker ps` shows strix containers still running after scan.
**Cause**: `session_manager.cleanup()` call raised (logged as exception in finally block); container left stranded.
**Recovery**:
```bash
docker container ls | grep strix
docker container rm -f <container-id>
```

---

## Known Gaps / Uncertainties

- Root cause for "agent produced non-lifecycle final output" (Failure 5) is provider-dependent — some LLMs call lifecycle tools less reliably than others.
- Caido bootstrap retry behavior under high-load environments not characterized.
