# PROMPTS

Parent document: /CLAUDE.md
Related documents:
- /docs/ai/AI_SYSTEM_MAP.md
- /docs/ai/LLM_ARCHITECTURE.md
- /docs/ai/MODEL_GUARDRAILS.md
- /docs/ai/AGENT_WORKFLOWS.md

Read this when:
- Making changes to agent behavior or instructions
- Adding or modifying skills
- Understanding what context agents receive

Purpose:
- Inventory of all prompts, their purpose, inputs, and output contracts

Scope:
- Prompts in this codebase only; not cloud platform prompts

---

## Prompt 1: Agent System Prompt

**File**: `strix/agents/prompts/system_prompt.jinja`
**Rendered by**: `strix/agents/prompt.py:render_system_prompt()`
**Purpose**: Defines all agent behavior — communication rules, testing methodology, tool usage, multi-agent coordination, vulnerability workflow, sandbox environment.

### Inputs to render()

| Variable | Type | Source |
|----------|------|--------|
| `loaded_skill_names` | list[str] | Resolved skill names |
| `available_skills` | dict[str, list[str]] | All user-selectable skills by category |
| `interactive` | bool | `interactive=True/False` from scan config |
| `system_prompt_context` | dict | `build_scope_context()` — authorized targets |
| `**skill_content` | kwargs | Loaded skill Markdown content, keyed by skill name |

### Key Sections in system_prompt.jinja

| Section | Purpose |
|---------|---------|
| `<core_capabilities>` | Agent self-description |
| `<communication_rules>` | Output format, interactive vs autonomous behavior rules |
| `<execution_guidelines>` | Scope enforcement, testing modes, methodology |
| `<vulnerability_focus>` | Priority vuln types, IDOR/SQLi/SSRF/XSS/etc. |
| `<multi_agent_system>` | Spawn rules, workflow patterns, agent roles |
| `<environment>` | Available sandbox tools (nmap, sqlmap, ffuf, nuclei, etc.) |
| `<specialized_knowledge>` | Injected skill content (conditional) |
| `<available_skills>` | List of loadable skills (conditional) |

### Scope Injection (security-critical)

When `system_prompt_context.authorized_targets` is non-empty:
```
SYSTEM-VERIFIED SCOPE: ...
AUTHORIZED TARGETS:
- web_application: https://example.com (workspace: /workspace/example-com)
```
This block explicitly instructs agents never to test targets outside this list. See `MODEL_GUARDRAILS.md`.

### Output Contract

The agent system prompt does not produce structured output directly. It instructs the agent to call tools, which produce structured JSON. The key lifecycle tool outputs are:
- `finish_scan` → `{"success": true, "scan_completed": true, "executive_summary": ..., ...}`
- `agent_finish` → `{"success": true, "agent_completed": true, ...}`

---

## Prompt 2: Deduplication Prompt

**File**: `strix/report/dedupe.py:DEDUPE_SYSTEM_PROMPT`
**Purpose**: LLM judge that decides whether a candidate vulnerability report is a duplicate of an existing report.

### Input

```json
{
  "candidate": {"title": "...", "description": "...", "target": "...", ...},
  "existing_reports": [{"id": "vuln-0001", "title": "...", ...}, ...]
}
```

### Expected Output Contract

```json
{
  "is_duplicate": true|false,
  "duplicate_id": "vuln-0001",
  "confidence": 0.95,
  "reason": "Both reports describe SQL injection in /api/login via the username parameter"
}
```

**Output validation**: `_parse_dedupe_response()` extracts JSON from response (handles markdown code fences), clamps `duplicate_id` to 64 chars, clamps `reason` to 500 chars.

**Failure behavior**: On any exception, returns `is_duplicate=False` — i.e., dedup failure allows the report through.

---

## Skills System

**Location**: `strix/skills/`
**Format**: Markdown files with optional YAML frontmatter (stripped before injection)
**Loading**: `strix/skills/__init__.py:load_skills()` reads files, strips frontmatter
**Injection**: Skills injected as Jinja2 `env.globals["get_skill"]` and template kwargs

### Always-Loaded Skills (system-managed)

| Skill | Path | When |
|-------|------|------|
| `scan_modes/<mode>` | `skills/scan_modes/deep.md` or `quick.md` | Always |
| `tooling/agent_browser` | `skills/tooling/agent_browser.md` | Always |
| `tooling/python` | `skills/tooling/python.md` | Always |
| `coordination/root_agent` | `skills/coordination/root_agent.md` | Root agent only |
| `coordination/source_aware_whitebox` | `skills/coordination/source_aware_whitebox.md` | White-box scans |
| `custom/source_aware_sast` | `skills/custom/source_aware_sast.md` | White-box scans |

### User-Selectable Skills (via `create_agent(skills=[...])` or `--skills` if supported)

Categories: `vulnerabilities/`, `frameworks/`, `technologies/`, `tooling/`, `cloud/`, `reconnaissance/`, `custom/`

Max 5 skills per agent. Internal categories (`scan_modes`, `coordination`) excluded from user selection.

---

## Known Gaps / Uncertainties

- Full skill file inventory not enumerated — many vulnerability skills assumed from the environment section listing them.
- Whether the `load_skill` tool can load internal skills (`scan_modes/`, `coordination/`) at runtime is unclear.
- `finish_scan` tool input/output schema not fully documented here — see `tools/finish/tool.py`.
