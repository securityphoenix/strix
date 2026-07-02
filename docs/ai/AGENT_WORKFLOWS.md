# AGENT_WORKFLOWS

Parent document: /CLAUDE.md
Related documents:
- /docs/ai/AI_SYSTEM_MAP.md
- /docs/ai/PROMPTS.md
- /docs/architecture/RUNTIME_FLOWS.md

Read this when:
- Understanding how multi-agent coordination works
- Debugging agent spawn/stop/messaging issues
- Designing new agent workflows

Purpose:
- Documents the multi-agent orchestration system, workflows, and tool contracts

Scope:
- Host-side coordination layer; sandbox tool execution is documented elsewhere

---

## Agent Graph Structure

Every scan has exactly one **root agent** (`strix`). The root orchestrates by spawning **child agents** with `create_agent`. Children can spawn their own children (nested trees). All share one Docker container, one `/workspace`, and one Caido proxy history.

```
strix (root)
    ├─ XSS Discovery Agent (skill: xss)
    │     └─ XSS Validation Agent (skill: xss)
    │           └─ XSS Reporting Agent (skill: xss)
    ├─ SQLi Agent (skill: sql_injection)
    │     └─ SQLi Validation Agent (skill: sql_injection)
    └─ Auth Specialist (skills: authentication_jwt, business_logic)
```

---

## Agent Lifecycle

| State | Meaning |
|-------|---------|
| `running` | Currently executing an LLM turn |
| `waiting` | Parked; waiting for an inbox message (interactive mode or `wait_for_message`) |
| `completed` | Called `agent_finish` or `finish_scan` successfully |
| `stopped` | Gracefully stopped by parent via `stop_agent` |
| `crashed` | Unhandled exception during execution |
| `failed` | SDK-level error (UserError, AgentsException, APIError) |

State transitions are written atomically to `agents.json` on every change via `_maybe_snapshot()`.

---

## Tools: Multi-Agent Graph

### `create_agent(name, task, inherit_context, skills)` → child spawned
- Spawns a new agent as an asyncio task running in parallel
- Child receives parent's turn input as background context if `inherit_context=True`
- Returns immediately with `agent_id` — parent continues without waiting
- Max 5 skills per agent

### `agent_finish(result_summary, findings, success, report_to_parent)` → subagent exits
- **Subagents only** — root must use `finish_scan`
- Posts structured completion report to parent's inbox
- Sets agent status to `completed`
- Stops agent execution (lifecycle tool)

### `finish_scan(executive_summary, methodology, technical_analysis, recommendations)` → scan ends
- **Root agent only** — subagents must use `agent_finish`
- Calls `report_state.update_scan_final_fields()` → triggers posthog.end(), disk write
- Sets `scan_completed=True` in output → `_lifecycle_tool_completed()` returns True → run exits

### `wait_for_message(reason, timeout_seconds)` → agent waits
- Parks agent until inbox has a message (or timeout, default 600s)
- In non-interactive mode: actual wait via `asyncio.wait_for`
- In interactive mode: returns immediately with `wait_outcome=waiting` (TUI drives wakeup)
- Lifecycle tool in interactive mode (causes `_wait_tool_parked` to return True → run exits and loop waits)

### `send_message_to_agent(target_agent_id, message, message_type, priority)` → message sent
- Injects message into target agent's SDK session
- Wakes the target if it's in `waiting` state
- In interactive mode: cancels target's current stream turn (immediate interrupt)

### `stop_agent(target_agent_id, cascade, reason)` → agent stopped
- Graceful: `cancel(mode="after_turn")` — current turn finishes before stopping
- `cascade=True` stops all descendants leaves-first

### `view_agent_graph()` → tree snapshot
- Returns indented tree showing all agents, their status, and parent relationships

---

## Prescribed Workflows (from system_prompt.jinja)

### Black-Box Workflow
```
Discovery Agent (finds potential vuln)
    → spawns Validation Agent (proves PoC works)
        → spawns Reporting Agent (calls create_vulnerability_report)
            [NO fixing agents in black-box]
```

### White-Box Workflow
```
Discovery Agent (static analysis)
    → spawns Validation Agent (dynamic exploit)
        → spawns Reporting Agent (create_vulnerability_report)
            → spawns Fixing Agent (patches code, verifies fix)
```

### Root Agent Role
Root agent is **coordinator, not tester**. It:
- Builds target map and initial plan
- Creates specialist subagents per vulnerability type × target component
- Monitors progress via `view_agent_graph`
- Routes specialist findings to validation/reporting chains
- Calls `finish_scan` with executive summary when all work is done

---

## Agent Messaging Protocol

Messages are appended to the target agent's SQLite session as user-role items:

- **From `user`**: `{"role": "user", "content": message}`
- **From another agent**: `{"role": "user", "content": "[Message from <name> (<id>) | type=... | priority=...]\n<content>"}`

The receiving agent sees these as additional context items in the next LLM turn.

---

## Human Approval Requirements

None. The system operates fully autonomously. The only human interaction points are:
- CLI arguments before scan start
- Interactive TUI (user can send messages to the root agent mid-scan)

---

## Known Gaps / Uncertainties

- Maximum practical agent count per scan not benchmarked — the system supports up to ~500 turns (DEFAULT_MAX_TURNS) for the root agent; child agent counts are bounded by available turns.
- What happens if a child agent outlives the root agent (root calls `finish_scan` while children are still running) is not fully documented — `cancel_descendants` is called in cleanup paths.
- Resume behavior for agents in `waiting` state vs `running` state differs (start_parked logic) but full behavior not documented.
