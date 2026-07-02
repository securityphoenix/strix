# LOCAL_DEVELOPMENT

Parent document: /CLAUDE.md
Related documents:
- /docs/development/TESTING_STRATEGY.md
- /docs/operations/RUNBOOK.md

Read this when:
- Setting up the development environment
- Running Strix from source
- Contributing code

Purpose:
- How to run locally, required env vars, setup order, common pitfalls

Scope:
- Local dev only; not CI/CD or production

---

## Prerequisites

- Python 3.12+
- [uv](https://docs.astral.sh/uv/) (fast Python package manager)
- Docker daemon running
- An LLM API key from any supported provider

---

## Setup

```bash
# 1. Clone
git clone https://github.com/usestrix/strix.git
cd strix

# 2. Install dev dependencies
make setup-dev
# or manually:
uv sync
uv run pre-commit install

# 3. Configure LLM
export STRIX_LLM="openai/gpt-5.4"
export LLM_API_KEY="your-api-key"

# Optional:
export PERPLEXITY_API_KEY="your-key"  # for web_search tool
export STRIX_REASONING_EFFORT="medium"  # lower cost for dev testing
```

---

## Running From Source

```bash
# Run a scan
uv run strix --target https://example.com

# Non-interactive (CI-style) scan
uv run strix -n --target ./test-app --scan-mode quick

# With debug logging (change configure_dependency_logging() or set Python log level)
PYTHONPATH=. python -c "import logging; logging.basicConfig(level=logging.DEBUG)" && uv run strix --target ...
```

---

## Quality Checks

```bash
# All checks (lint + types + security)
make check-all

# Individual checks
uv run ruff check strix/         # linting
uv run ruff format --check strix/ # formatting
uv run mypy strix/               # strict type checking
uv run pyright strix/            # alternative type checker
uv run bandit -r strix/          # security linting

# Auto-fix formatting + imports
uv run ruff format strix/
uv run ruff check --fix strix/
```

---

## Running Tests

```bash
# No formal test suite found in this repo
# Tests (if present) would be in benchmarks/ or a tests/ directory
# Functional testing is done by running actual scans
uv run strix -n --target <safe-test-target> --scan-mode quick
```

---

## Pre-commit Hooks

```bash
# Installed via uv run pre-commit install
# Runs on git commit: ruff, mypy, and other checks
# Skip for emergency:
git commit --no-verify
```

---

## Building the PyInstaller Binary

```bash
# Requires Python 3.12-3.14
uv run pyinstaller strix.spec
# Output: dist/strix
```

---

## Common Pitfalls

| Problem | Cause | Fix |
|---------|-------|-----|
| `STRIX_LLM not set` error | Missing env var | `export STRIX_LLM="openai/gpt-5.4"` |
| Docker not found | Docker Desktop not running | Start Docker daemon |
| LLM warmup timeout | Slow provider or wrong credentials | Check API key; increase `LLM_TIMEOUT` |
| `uv run strix` not found | Wrong directory | Must be in repo root |
| Mypy errors on third-party libs | Missing stubs | Expected; `ignore_missing_imports = true` for listed libs |
| `ruff` format issues | Auto-fixable | Run `uv run ruff format strix/` |

---

## Environment Variable Reference (Dev)

| Var | Recommended Value | Why |
|-----|------------------|-----|
| `STRIX_LLM` | `anthropic/claude-sonnet-4-6` or `openai/gpt-5.4` | Required |
| `LLM_API_KEY` | your actual key | Required for most providers |
| `STRIX_REASONING_EFFORT` | `medium` | Reduces cost during development |
| `STRIX_IMAGE` | `ghcr.io/usestrix/strix-sandbox:1.0.0` | Default; override for custom sandbox |
| `STRIX_TELEMETRY` | `false` | Disable telemetry during dev |
| `LLM_TIMEOUT` | `120` | Faster failure in dev |

---

## Known Gaps / Uncertainties

- No formal unit test suite found in this repo.
- Makefile targets beyond `setup-dev` and `check-all` not documented here.
- How to build the sandbox Docker image (`ghcr.io/usestrix/strix-sandbox`) is in a separate repo.
