# TESTING_STRATEGY

Parent document: /CLAUDE.md
Related documents:
- /docs/development/LOCAL_DEVELOPMENT.md
- /docs/development/SAFE_CHANGE_ZONES.md

Read this when:
- Writing tests for new features
- Understanding what is and isn't tested
- Assessing test coverage gaps

Purpose:
- Documents testing approach, what tests exist, and what coverage reveals

Scope:
- This codebase only; not cloud platform tests

---

## Test Coverage Status

**No formal unit test suite was found in this repository.** The `benchmarks/` directory exists but was not analyzed.

Existing quality assurance:
- **Static type checking**: `mypy` (strict) + `pyright` (strict) — catches type errors
- **Linting**: `ruff` (comprehensive rule set including security rules)
- **Security linting**: `bandit` (medium severity threshold)
- **Pre-commit hooks**: Enforce all of the above on commit

---

## What Pre-Commit / CI Checks Cover

| Check | What It Catches |
|-------|----------------|
| `ruff check` | Import order, style, unused variables, common bugs, security patterns (S rules) |
| `ruff format` | Code formatting |
| `mypy --strict` | Type annotation correctness, missing types, unreachable code |
| `pyright strict` | Alternative type checking (catches different issues) |
| `bandit` | Security anti-patterns (subprocess, hardcoded secrets, etc.) |

---

## Functional Testing Approach

Since there is no unit test suite, functional testing is done by running actual scans:

```bash
# Minimal smoke test (non-interactive, safe target)
uv run strix -n --target https://example.com --scan-mode quick

# Expected: exits with 0 (no findings) or 2 (findings found)
# Unexpected: Python exception, Docker error, hang
```

**For testing specific features**:
- **Resume**: Run a scan, Ctrl+C mid-scan, then `strix --resume <run-name>`
- **White-box**: `strix -n --target ./test-app --scan-mode quick`
- **New tool**: Run a scan that exercises the tool path, check log output
- **Prompt changes**: Run a scan and verify agent behavior in log + output

---

## What Tests Reveal (from type system)

The strict type checking setup reveals:
- All tool function signatures are typed and validated at import time
- `RunContextWrapper`, `Tool` types are imported eagerly (not under `TYPE_CHECKING`) for SDK JSON schema derivation
- Lazy imports in tool files are explicitly documented in `pyproject.toml` ruff ignores

---

## Recommended Test Areas (Not Yet Covered)

| Area | Why Important | Suggested Approach |
|------|--------------|-------------------|
| `tools/reporting/tool.py` validation | Strict field validation; bugs silently drop vulns | Unit tests for `_do_create()` edge cases |
| `report/dedupe.py` | LLM-based; hard to test without mocking | Mock `MultiProvider().get_model()` |
| `core/agents.py:AgentCoordinator` | Async state machine; resume correctness | asyncio unit tests with mock agents |
| `config/loader.py` | Env/file precedence logic | Unit tests with mock env vars |
| `interface/utils.py:infer_target_type()` | Target type detection for all formats | Parametrized unit tests |
| `report/writer.py` | Disk write correctness for resume | tmpdir-based unit tests |

---

## Known Gaps / Uncertainties

- `benchmarks/` directory content not analyzed — may contain performance benchmarks.
- No contract tests for Caido GraphQL API.
- No prompt/schema tests for AI outputs.
- No integration tests for the full scan pipeline (would require Docker + LLM API).
