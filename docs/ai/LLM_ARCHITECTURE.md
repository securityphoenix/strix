# LLM_ARCHITECTURE

Parent document: /CLAUDE.md
Related documents:
- /docs/ai/AI_SYSTEM_MAP.md
- /docs/ai/PROMPTS.md
- /docs/integrations/EXTERNAL_INTEGRATIONS.md

Read this when:
- Changing LLM provider or model
- Debugging model connectivity issues
- Understanding request pipeline and retry behavior

Purpose:
- LLM provider integration, request pipeline, retries, fallbacks, tool schema handling

Scope:
- openai-agents SDK integration layer only

---

## Model Resolution

```
STRIX_LLM=anthropic/claude-sonnet-4-6
         │
         ▼
config/models.py:normalize_model_name()
  → strips prefix (e.g., "anthropic/") for SDK routing
         │
         ▼
agents.models.multi_provider.MultiProvider().get_model(resolved_name)
  → routes to LiteLLM or provider-native client based on model name
         │
         ▼
openai-agents SDK streams model responses
```

---

## Request Pipeline (per agent turn)

```
Runner.run_streamed(agent, input, run_config, context, session, hooks)
    │
    ├─ system_prompt = agent.instructions (pre-rendered Jinja2 string)
    ├─ tools = agent.tools (list of FunctionTool + CustomTool)
    ├─ model_settings:
    │     parallel_tool_calls: False
    │     tool_choice: "required" (if no reasoning) OR None (if reasoning enabled)
    │     retry: DEFAULT_MODEL_RETRY (value in config/models.py — not inspected)
    │     include_usage: True
    │     reasoning: Reasoning(effort=STRIX_REASONING_EFFORT) if reasoning enabled
    │
    ├─ SDK assembles messages from session history (SQLite) + initial_input
    └─ streams response events: tool_call → tool_result → final_output
```

---

## Tool Schema Compatibility

Two modes based on `uses_chat_completions_tool_schema()` in `config/models.py`:

| Mode | Applies To | Tool Format |
|------|-----------|------------|
| Responses (native) | Most providers | SDK CustomTool + FunctionTool as-is |
| Chat Completions | Some providers | CustomTools wrapped as FunctionTools via `_custom_tool_as_function_tool()` |

The wrapping in `factory.py:_configure_chat_completions_filesystem_tools()` converts SDK custom tools (filesystem, shell) into plain function tools with a single string `input` field when the provider doesn't support the Responses API format.

---

## Reasoning Effort

Controlled by `STRIX_REASONING_EFFORT` (default: `high`):
- Allowed values: `none`, `minimal`, `low`, `medium`, `high`, `xhigh`
- When non-`none`: `tool_choice=None` (models reject `tool_choice="required"` with reasoning enabled)
- `make_model_settings()` in `core/inputs.py` applies this

**Known issue fixed in v1.0.2**: Reasoning models (Anthropic/DeepSeek) reject `tool_choice="required"`. The fix sets `tool_choice=None` when reasoning is enabled, relying on prompt discipline and `_finish_tool_use_behavior` to keep the loop converging.

---

## Retries and Error Recovery

**SDK-level retry**: `ModelSettings(retry=DEFAULT_MODEL_RETRY)` — retry count not confirmed but present.

**Image-strip retry** (`execution.py:_run_cycle`):
- Triggered on: API error with status_code in `{400, 404, 422}` when a session exists
- Action: `strip_latest_image_from_session()` removes the most recent image from session history
- Limit: 3 strips per cycle
- Purpose: Handles context-window exhaustion from large image attachments

**Non-interactive tool-required recovery** (`execution.py:_run_noninteractive_until_lifecycle`):
- Triggered when: LLM produces text-only output without a lifecycle tool call
- Action: Injects a user message demanding a tool call
- Limit: `max(1, max_turns)` attempts (default 500) before `MaxTurnsExceeded`

---

## Fallbacks

| Scenario | Behavior |
|---------|---------|
| LLM warm-up fails | `sys.exit(1)` — scan does not start |
| Agent turn LLM failure (interactive) | Agent parked as `failed`; parent notified |
| Agent turn LLM failure (non-interactive) | Exception propagates to `run_strix_scan`, scan aborted |
| Dedupe LLM call fails | `is_duplicate=False` returned — vulnerability filed anyway |
| CVSS lib fails | Score defaults to 7.5/high |

---

## Supported Providers (via LiteLLM)

OpenAI, Anthropic, Google (Gemini), Vertex AI, AWS Bedrock, Azure OpenAI, Ollama, LMStudio, any OpenAI-compatible endpoint.

Provider selection via `STRIX_LLM` format: `<provider>/<model>` or just `<model>` for OpenAI-native.

---

## Known Gaps / Uncertainties

- `DEFAULT_MODEL_RETRY` value in `config/models.py` not confirmed.
- `normalize_model_name()` full transformation logic not analyzed.
- Whether LiteLLM caches connections between agent turns is unknown.
- Token limit handling (context window exhaustion beyond image strips) is not documented.
