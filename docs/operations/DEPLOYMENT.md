# DEPLOYMENT

Parent document: /CLAUDE.md
Related documents:
- /docs/development/LOCAL_DEVELOPMENT.md
- /docs/integrations/EXTERNAL_INTEGRATIONS.md

Read this when:
- Building and releasing a new version
- Understanding the CI/CD pipeline
- Packaging the CLI for distribution

Purpose:
- Build, release, and deployment process

Scope:
- CLI tool distribution; not cloud platform deployment

---

## Distribution Formats

| Format | How Distributed | Notes |
|--------|----------------|-------|
| PyPI package | `pip install strix-agent` | Pure Python, requires Python 3.12+ |
| PyInstaller binary | GitHub Releases | Self-contained binary, no Python required |
| Docker image (sandbox) | `ghcr.io/usestrix/strix-sandbox` | Kali Linux sandbox; not the CLI itself |
| Install script | `curl -sSL https://strix.ai/install \| bash` | Installs binary from GitHub Releases |

---

## CI/CD Pipeline

**File**: `.github/workflows/build-release.yml`
**Triggers**: Releases (tag-based)

**Build steps** (inferred from `pyproject.toml` + `scripts/`):
1. Matrix build across Python 3.12/3.13/3.14 + platforms (Linux, macOS, Windows)
2. `uv sync` — install dependencies
3. Linting: `ruff check`, `ruff format --check`
4. Type checking: `mypy` (strict), `pyright` (strict)
5. Security linting: `bandit`
6. `pyinstaller strix.spec` — build standalone binary
7. Publish to PyPI
8. Publish binary to GitHub Releases

---

## Building Locally

```bash
# Install dev dependencies
make setup-dev
# or:
uv sync

# Run quality checks
make check-all

# Build PyInstaller binary
uv run pyinstaller strix.spec

# Build Docker sandbox image (scripts/docker.sh)
./scripts/docker.sh
```

---

## Versioning

**File**: `pyproject.toml:version`
**Current**: `1.0.2`
**Scheme**: semver-like; no automated bumping observed

**Notable version history**:
- `1.0.2`: Fix reasoning models rejecting `tool_choice="required"`; fix PyInstaller bundle
- `1.0.1`: Fix PyInstaller bundle (missing agents SDK data, wrongly excluded gql)
- `1.0.0`: Initial stable release

---

## Runtime Environment (Production)

Strix is a CLI tool installed on user machines. "Production" is each user's machine + Docker daemon.

**Runtime requirements**:
- Python 3.12+ OR PyInstaller binary (no Python required)
- Docker daemon running locally
- `STRIX_LLM` env var set
- Outbound HTTPS to LLM provider
- Outbound Docker pull from GHCR (first run only)

---

## Sandbox Image

**Image**: `ghcr.io/usestrix/strix-sandbox:1.0.0`
**Registry**: GitHub Container Registry (`ghcr.io/usestrix/`)
**Override**: `STRIX_IMAGE` env var
**Contents**: Kali Linux + security tools (nmap, sqlmap, ffuf, nuclei, semgrep, Caido CLI, etc.)
**Build**: Not in this repo — separate repository/process

---

## Known Gaps / Uncertainties

- Full `.github/workflows/build-release.yml` content not analyzed.
- Sandbox image build process is in a separate repository (not this one).
- No staging environment described — users run against their own targets.
