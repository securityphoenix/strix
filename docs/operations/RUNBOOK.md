# RUNBOOK

Parent document: /CLAUDE.md
Related documents:
- /docs/operations/FAILURE_MODES.md
- /docs/operations/OBSERVABILITY.md
- /docs/development/LOCAL_DEVELOPMENT.md

Read this when:
- Running, interrupting, or resuming scans
- Performing routine maintenance
- Operating the system in CI/CD environments

Purpose:
- Common operational tasks for running and maintaining Strix

Scope:
- CLI operations; not cloud platform operations

---

## Running a Scan

### Basic scan
```bash
export STRIX_LLM="openai/gpt-5.4"
export LLM_API_KEY="sk-..."
strix --target https://app.example.com
```

### CI/CD mode (non-interactive, exits 2 if findings)
```bash
strix -n --target ./ --scan-mode quick
```

### White-box scan with source + live endpoint
```bash
strix --target ./app-directory --target https://staging.example.com
```

### Authenticated test with instructions
```bash
strix --target https://app.example.com \
  --instruction "Test as authenticated user: admin/password123. Focus on IDOR."
```

---

## Resuming an Interrupted Scan

```bash
# Find prior run names
ls strix_runs/

# Resume from checkpoint
strix --resume web-application-abc123

# Resume with additional instruction
strix --resume web-application-abc123 \
  --instruction "Focus on the /api/admin endpoints we missed earlier."
```

**Requirements for resume**:
- `strix_runs/<run>/run.json` must exist
- `strix_runs/<run>/.strix_state/agents.json` must exist
- `strix_runs/<run>/.strix_state/agents.db` must exist
- If target was a git repo: cloned path must still exist

---

## Checking Scan Output

```bash
# Run directory structure
strix_runs/<run-name>/
├── run.json                    # Scan metadata and status
├── vulnerabilities.json        # All vulnerability reports
├── vuln-0001.md               # Individual vulnerability Markdown
├── executive_report.md         # Final executive summary (if completed)
├── strix-<run>.log            # Per-scan log file
└── .strix_state/
    ├── agents.json             # Agent graph snapshot
    ├── agents.db               # SDK conversation history (SQLite)
    ├── notes.json              # Agent notes (if any)
    └── todos.json              # Agent todos (if any)
```

---

## Interrupting a Scan

- **Interactive mode**: Ctrl+C or close TUI → `signal_handler` → `report_state.cleanup(status="interrupted")` → run.json status = "interrupted"
- **Non-interactive mode**: Ctrl+C → same path
- **Resume after interrupt**: `strix --resume <run-name>`

**Note**: The scan state is safe to interrupt at any point — the coordinator snapshot is written atomically on every state change, so resume picks up where the agents left off.

---

## Changing Configuration

```bash
# One-time override
export STRIX_LLM="anthropic/claude-sonnet-4-6"
strix --target ...  # New model used, persisted to ~/.strix/cli-config.json

# Custom config file
strix --config /path/to/custom-config.json --target ...

# Custom config JSON format:
# {"env": {"STRIX_LLM": "openai/gpt-5.4", "LLM_API_KEY": "sk-..."}}
```

---

## Docker Image Management

```bash
# Pull latest image manually
docker pull ghcr.io/usestrix/strix-sandbox:1.0.0

# Override image (e.g., for development)
export STRIX_IMAGE="my-custom-sandbox:latest"

# Clean up stopped scan containers (if Strix crashed without cleanup)
docker container ls -a | grep strix
docker container rm <id>
```

---

## Viewing Scan Logs

```bash
# Per-scan log (most useful for debugging)
tail -f strix_runs/<run-name>/strix-<run-name>.log

# Python logging level can be changed by modifying configure_dependency_logging()
# in strix/telemetry/logging.py (no env var control currently)
```

---

## Known Gaps / Uncertainties

- No log rotation for per-scan logs — large scans can produce large log files.
- `configure_dependency_logging()` behavior (what logging levels it sets) not fully analyzed.
- No built-in tool to list or summarize all runs (users must inspect `strix_runs/` manually).
