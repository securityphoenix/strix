# SAFE_CHANGE_ZONES

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/CHANGE_BLAST_RADIUS.md
- /docs/development/TESTING_STRATEGY.md

Read this when:
- About to modify any module and assessing risk
- Deciding where to put new code
- Code reviewing a PR

Purpose:
- Classifies modification risk and mandatory checks per area

Scope:
- Source code change risk; not deployment risk

---

## Safe to Change (Low Risk)

These areas can be changed with normal care and a scan smoke test.

| Area | Notes |
|------|-------|
| `strix/skills/**/*.md` — adding new skill files | Auto-discovered; can only add capabilities |
| `strix/telemetry/posthog.py`, `scarf.py` | Fire-and-forget; no functional impact |
| `strix/interface/utils.py:generate_run_name()` | Only affects output directory naming |
| `strix/core/paths.py` | Path conventions; verify all callers use updated paths |
| `strix/report/usage.py` | Token usage accounting; cosmetic impact |
| `pyproject.toml` — linting/formatting config | No runtime impact |
| `README.md`, `CONTRIBUTING.md` — docs | Documentation only |

---

## Change Carefully (Medium Risk)

These areas require reading the referenced docs before touching.

| Area | Mandatory Read Before | Risk |
|------|--------------------|------|
| `strix/interface/main.py` — startup flow | `RUNTIME_FLOWS.md` | Pre-flight checks ordering matters |
| `strix/interface/utils.py` — target inference | `DATA_CONTRACTS.md` | `target_type` values used in prompts and run.json |
| `strix/config/loader.py` | `DATA_CONTRACTS.md` | Settings precedence and persistence |
| `strix/runtime/session_manager.py` | `OPERATIONAL_FAILURE_GRAPH.md` | Container lifecycle; cleanup must remain best-effort |
| `strix/tools/reporting/tool.py` | `DATA_CONTRACTS.md` | Vulnerability schema validation |
| `strix/tools/agents_graph/tools.py` | `AGENT_WORKFLOWS.md` | Tool contracts for multi-agent coordination |
| `strix/report/writer.py` | `DATA_LINEAGE.md` | On-disk format must remain resume-compatible |
| `strix/agents/prompt.py` | `PROMPTS.md` | Skill loading and ordering affects all agents |
| `strix/skills/__init__.py` | `PROMPTS.md` | Skill discovery and frontmatter stripping |

---

## High Risk — Treat as Critical Path

Read ALL referenced docs. Test interactive AND non-interactive modes. Test resume.

| Area | Mandatory Reads | Specific Risks |
|------|----------------|----------------|
| `strix/agents/prompts/system_prompt.jinja` | `PROMPTS.md`, `MODEL_GUARDRAILS.md`, `AGENT_WORKFLOWS.md` | All agent behavior; scope enforcement |
| `strix/core/execution.py` | `RUNTIME_FLOWS.md`, `CALL_GRAPH.md`, `CHANGE_BLAST_RADIUS.md` | Agent loop, resume, image-strip retry |
| `strix/core/agents.py` | `AGENT_WORKFLOWS.md`, `DATA_CONTRACTS.md` | Coordinator state, snapshot atomicity |
| `strix/report/state.py` | `DATA_LINEAGE.md`, `DATA_CONTRACTS.md` | Global singleton, write ordering |
| `strix/core/runner.py` | `RUNTIME_FLOWS.md`, `RUNTIME_DEPENDENCY_TREE.md` | Scan orchestration, resume path |
| `strix/agents/factory.py` | `AGENT_WORKFLOWS.md`, `CHANGE_BLAST_RADIUS.md` | Tool list, lifecycle tool detection |
| `strix/runtime/caido_bootstrap.py` | `RUNTIME_DEPENDENCY_TREE.md` | Caido readiness retry |

---

## Mandatory Checks Before Modifying Critical Modules

### Before changing `system_prompt.jinja`
- [ ] Read `PROMPTS.md` and `MODEL_GUARDRAILS.md`
- [ ] Verify `authorized_targets` scope block is intact and correct
- [ ] Test in both interactive (`run_tui`) and non-interactive (`-n`) modes
- [ ] Verify lifecycle tool calls still work (finish_scan / agent_finish)
- [ ] Test that agent calls tools (not text-only output) in autonomous mode

### Before changing `execution.py`
- [ ] Read `RUNTIME_FLOWS.md` fully
- [ ] Test resume: run scan → Ctrl+C → `strix --resume`
- [ ] Test non-interactive scan completes with `finish_scan`
- [ ] Test child agent spawn and `agent_finish` delivery to parent
- [ ] Verify image-strip retry path is not broken

### Before changing `agents.py` (AgentCoordinator)
- [ ] Verify `_maybe_snapshot()` remains atomic (temp file + rename)
- [ ] Verify `_lock` protects all mutable state
- [ ] Test resume with multiple child agents in different states

### Before changing `state.py` (ReportState)
- [ ] Verify `add_vulnerability_report()` still allocates sequential IDs
- [ ] Verify `save_run_data()` does not throw (would break agent turns)
- [ ] Test that `vulnerability_found_callback` is called (live UI update)

---

## Known Gaps / Uncertainties

- No automated test suite to validate safe change zones — all verification is manual scan testing.
- TUI-specific code paths not covered by any known automated checks.
