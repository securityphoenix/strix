# REPOSITORY_MAP

Parent document: /CLAUDE.md
Related documents:
- /docs/architecture/MODULE_OWNERSHIP.md
- /docs/architecture/DEPENDENCY_GRAPH.md
- /docs/architecture/RUNTIME_FLOWS.md

Read this when:
- Navigating the codebase for the first time
- Looking for where a specific capability lives
- Understanding how directories relate to each other

Purpose:
- Maps every major directory and its role

Scope:
- Source layout only; not runtime behavior

---

## Top-Level Directory

```
strix-phx/
├── strix/                  # Python package (installed as strix-agent)
│   ├── interface/          # User-facing entry points (CLI, TUI, argument parsing)
│   ├── core/               # Scan runner, agent loop, coordinator, inputs
│   ├── agents/             # Agent factory + system prompt renderer
│   ├── tools/              # Host-side function tools registered with SDK
│   ├── config/             # Settings model, loader, persistence
│   ├── runtime/            # Docker sandbox, Caido bootstrap, session lifecycle
│   ├── report/             # Scan artifact state, writer, deduplication
│   ├── skills/             # Markdown skill files (injected into prompts)
│   ├── telemetry/          # PostHog + Scarf telemetry
│   └── utils/              # Resource path helpers
├── docs/                   # Documentation (this tree)
├── benchmarks/             # Performance benchmarks (not analyzed)
├── containers/             # Docker entrypoint script
├── scripts/                # Build, install, Docker helper scripts
├── .github/workflows/      # CI/CD pipeline (build-release.yml)
├── pyproject.toml          # Package config, dependencies, tool config
├── strix.spec              # PyInstaller bundle spec
└── uv.lock                 # Locked dependency tree
```

---

## strix/interface/

**Role**: User entry points. Owns argument parsing, pre-flight checks, UI rendering.

```
interface/
├── main.py         # Binary entry point (strix = strix.interface.main:main)
│                   # parse_arguments, validate_environment, pull_docker_image,
│                   # warm_up_llm, clone_repository, run_cli/run_tui dispatch
├── cli.py          # Headless/non-interactive run_cli() — Rich console output
├── utils.py        # Target type inference, diff scope, repo clone, run name gen
└── tui/            # Interactive Textual TUI (not fully analyzed)
    └── app.py      # Main TUI application
```

**Where to start**: `main.py:main()` — sequential startup: docker check → env validate → LLM warmup → run cli/tui.

---

## strix/core/

**Role**: Core scan mechanics. Owns the scan execution contract.

```
core/
├── runner.py       # run_strix_scan() — top-level scan orchestrator
│                   # Creates sandbox, builds agent, runs loop, handles resume
├── execution.py    # run_agent_loop(), spawn_child_agent(), respawn_subagents()
│                   # The asyncio event loop driving SDK streaming runs
├── agents.py       # AgentCoordinator — graph state, messages, snapshots
│                   # AgentRuntime — per-agent session/task/stream refs
├── inputs.py       # build_root_task(), child_initial_input(), make_model_settings()
│                   # Pure input construction, no side effects
├── sessions.py     # open_agent_session(), strip_latest_image_from_session()
│                   # SQLite session management via SDK
├── hooks.py        # ReportUsageHooks — records token usage from SDK events
└── paths.py        # run_dir_for(), runtime_state_dir() — path conventions
```

**Where to start**: `runner.py:run_strix_scan()` — this is the programmatic API surface.

---

## strix/agents/

**Role**: Agent construction and system prompt rendering.

```
agents/
├── factory.py      # build_strix_agent(), make_child_factory()
│                   # Tool list assembly, capability config, lifecycle tool behavior
├── prompt.py       # render_system_prompt() — Jinja2 template rendering
│                   # _resolve_skills() — skill list ordering
└── prompts/
    └── system_prompt.jinja   # Master prompt template (all agent behavior)
```

---

## strix/tools/

**Role**: Host-side function tools made available to agents via SDK registration.

```
tools/
├── agents_graph/tools.py    # create_agent, stop_agent, view_agent_graph,
│                            # send_message_to_agent, wait_for_message, agent_finish
├── finish/tool.py           # finish_scan (root agent lifecycle tool)
├── load_skill/tool.py       # load_skill (pull skill inline during run)
├── notes/tools.py           # create_note, list_notes, get_note, update_note, delete_note
├── proxy/tools.py           # list_requests, view_request, repeat_request,
│                            # list_sitemap, view_sitemap_entry, scope_rules
├── proxy/caido_api.py       # Caido SDK client wrapper
├── reporting/tool.py        # create_vulnerability_report (files a vuln finding)
├── thinking/tool.py         # think (private reasoning scratch-pad)
├── todo/tools.py            # create_todo, list_todos, mark_todo_done, etc.
└── web_search/tool.py       # web_search (Perplexity API or fallback)
```

**Note**: Sandbox-bound Shell and Filesystem tools are emitted by the SDK's `Shell` and `Filesystem` capabilities in `factory.py`, not imported from here.

---

## strix/config/

**Role**: Settings resolution and persistence.

```
config/
├── settings.py     # Pydantic-settings models: LlmSettings, RuntimeSettings,
│                   # TelemetrySettings, IntegrationSettings, Settings
├── loader.py       # load_settings() (memoized), persist_current(), apply_config_override()
│                   # Precedence: env vars > ~/.strix/cli-config.json > field defaults
└── models.py       # configure_sdk_model_defaults(), normalize_model_name(),
                    # uses_chat_completions_tool_schema() (not fully analyzed)
```

---

## strix/runtime/

**Role**: Container lifecycle and proxy bootstrap.

```
runtime/
├── session_manager.py    # create_or_reuse(), cleanup() — per-scan container cache
├── backends.py           # Backend registry: currently only "docker"
│                         # register_backend() for custom runtime injection
├── docker_client.py      # StrixDockerSandboxClient — adds NET_ADMIN/NET_RAW caps
│                         # + host.docker.internal gateway to container
└── caido_bootstrap.py    # bootstrap_caido() — guest login, project creation
```

---

## strix/report/

**Role**: Scan artifact ownership and persistence.

```
report/
├── state.py        # ReportState (global singleton per scan)
│                   # add_vulnerability_report(), save_run_data(), cleanup()
│                   # vulnerability_found_callback hook for live UI updates
├── writer.py       # write_run_record(), write_vulnerabilities(),
│                   # write_executive_report(), read_run_record()
├── dedupe.py       # check_duplicate() — LLM-based deduplication call
├── usage.py        # LLMUsageLedger — per-agent token tracking
└── state.py        # (also holds get/set_global_report_state singletons)
```

---

## strix/skills/

**Role**: Markdown knowledge files injected into agent system prompts.

```
skills/
├── vulnerabilities/    # Vuln-specific payloads and guidance (e.g., xss, sqli, ssrf)
├── frameworks/         # Framework-specific testing (not listed, assumed present)
├── technologies/       # Technology-specific (supabase, firebase_firestore)
├── tooling/            # Tool usage guides (nmap, nuclei, sqlmap, ffuf, semgrep, etc.)
├── cloud/              # Cloud-specific (kubernetes)
├── reconnaissance/     # Recon skill files
├── coordination/       # root_agent, source_aware_whitebox (internal, not user-selectable)
├── scan_modes/         # deep.md, quick.md (internal, not user-selectable)
└── custom/             # Custom skill files
```

Internal skill categories (`coordination`, `scan_modes`) are not user-selectable via `--skills` CLI arg.

---

## Known Gaps / Uncertainties

- `strix/interface/tui/` subdirectory structure not fully mapped.
- `benchmarks/` not analyzed.
- Complete skill file inventory not enumerated (many vulnerability skills assumed but not verified).
- `strix/tools/proxy/caido_api.py` content not fully read.
