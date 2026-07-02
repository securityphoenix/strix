# Strix Vulnerability Methodology

**Purpose**: Engineering reference for implementing parity with Strix's approach to finding, validating, and remediating vulnerabilities using a multi-agent system.

**Sources**: Derived directly from `system_prompt.jinja`, `scan_modes/`, `coordination/`, and `vulnerabilities/` skill files. Every statement here is backed by actual system behavior, not documentation assumptions.

---

## 1. Fundamental Principle

**No report without validation.** Static analysis output is a hypothesis until a working proof-of-concept is demonstrated. This is the central invariant that drives every design decision in the methodology.

The scoring contract is equally strict: a vulnerability is only considered reported when `create_vulnerability_report` is called with a working PoC script (`poc_script_code` is required, not optional). Mentions in completion messages or summaries do not count.

---

## 2. Scan Modes

Three modes exist. The mode is injected into the system prompt and shapes agent behavior throughout the scan.

### Quick Mode (CI/PR scans)

**Goal**: Fast feedback on high-impact issues. Prioritize breadth over depth.

**Phase 1 — Rapid Orientation**
- *Whitebox*: Focus on **recent changes** — git diffs, new commits, modified files. Run fast static triage on changed files first (`semgrep`, targeted `sg` queries). At least one lightweight AST pass (scoped to changed/high-risk paths only). Quick secrets/deps check (`gitleaks`, `trufflehog`, `trivy fs`) on changed areas.
- *Blackbox*: Map authentication and critical user flows. Skip deep content discovery. Test what is immediately accessible.

**Phase 2 — High-Impact Targets (priority order)**
1. Authentication bypass — login flaws, session issues, token weaknesses
2. Broken access control — IDOR, privilege escalation, missing authorization
3. Remote code execution — command injection, deserialization, SSTI
4. SQL injection — authentication endpoints, search, filters
5. SSRF — URL parameters, webhooks, integrations
6. Exposed secrets — hardcoded credentials, API keys, config files

**Explicitly skipped**: exhaustive subdomain enumeration, full directory bruteforcing, low-severity info disclosure, theoretical issues without working PoC.

**Agent count**: Subagents created only for parallel high-priority tasks. No full hierarchy for minor tasks.

**Mindset**: Time-boxed bug bounty hunter. Validate quickly and move on. If an attack vector isn't yielding results, pivot.

---

### Standard Mode (routine testing)

**Goal**: Balanced coverage with structured methodology. Understand before exploiting.

**Phase 1 — Reconnaissance**
- *Whitebox*: Map codebase (modules, entry points, routing). `semgrep` first-pass triage. At least one AST-structural pass (`sg` and/or Tree-sitter) for route/sink/trust-boundary mapping. Trace input vectors. Review auth/authz flows. Analyze DB interactions. Check deps with `trivy fs`, `gitleaks`, `trufflehog`.
- *Blackbox*: Crawl thoroughly, interact with every feature. Enumerate endpoints. Fingerprint stack. Capture traffic via proxy.

**Phase 2 — Business Logic Analysis**
- Map critical flows (payments, registration, data access, admin)
- Identify role boundaries
- Document data access rules and isolation
- Trace state transitions
- Identify trust boundaries

**Phase 3 — Systematic Testing** (spawn focused subagents per area)
- Input validation: injection on all fields (SQLi, XSS, command, template), file upload bypass, search/filter manipulation
- Auth & session: brute force protection, token entropy, password reset flow, logout invalidation, auth bypass
- Access control: horizontal (user A → user B resources), vertical (unpriv → admin), API vs UI consistency, direct object references
- Business logic: multi-step bypass, race conditions, boundary conditions, transaction replay

**Phase 4 — Exploitation**: Working PoC for every finding. Chain vulns for maximum severity. Full attack path documented.

**Phase 5 — Reporting**: All confirmed vulns, severity by exploitability + business impact, remediation, areas for further investigation.

---

### Deep Mode (full penetration test, default)

**Goal**: Exhaustive coverage. Find what others miss.

**Phase 1 — Exhaustive Reconnaissance**
- *Whitebox*:
  - Map every file, module, and code path
  - Broad source-aware triage: `semgrep`, `ast-grep`, `gitleaks`, `trufflehog`, `trivy fs`
  - At least one structural AST pass (`sg` and/or Tree-sitter) per repository; store artifacts for reuse
  - Trace all entry points from HTTP handlers to database queries
  - Document all auth mechanisms, authz checks, external integrations
  - Analyze config for secrets and misconfigs
  - Review database schemas, background jobs, cron tasks, async processing
  - Map all serialization/deserialization points
  - Review all file handling (upload, download, processing)
  - Check all dependency versions against CVE data
- *Blackbox*:
  - Exhaustive subdomain enumeration (multiple sources/tools)
  - Full port scanning
  - Complete content discovery (multiple wordlists)
  - Technology fingerprinting on all assets
  - API discovery via docs, JS analysis, fuzzing
  - Identify all parameters including hidden/rarely-used
  - Map all user roles
  - Document rate limiting, WAF rules, security controls

**Phase 2 — Business Logic Deep Dive**
- Complete storyboard of the application
- User flows: every step of every workflow
- State machines: all transitions
- Trust boundaries: where privilege changes hands
- Invariants: what rules must always hold
- Implicit assumptions: what can be violated
- Multi-step attack surfaces
- Third-party integration map

**Phase 3 — Comprehensive Attack Surface Testing**
- Input handling: multiple injection types + encoding bypasses + boundary conditions
- Auth & session: brute force, session fixation/hijacking/prediction, JWT, OAuth flow abuse, password reset, MFA bypass, account enumeration
- Access control: every endpoint for horizontal + vertical access, parameter tampering, forced browsing, HTTP method tampering, post-state-change access
- File operations: upload bypass (extension + content-type + magic bytes), path traversal, SSRF via file inclusion, XXE
- Business logic: race conditions, workflow bypass, price/quantity manipulation, parallel execution, TOCTOU
- Advanced: HTTP request smuggling, cache poisoning, subdomain takeover, prototype pollution, CORS misconfiguration, WebSocket security, GraphQL attacks

**Phase 4 — Vulnerability Chaining**
Treat every finding as a pivot point. Chain to maximum impact:
- Info disclosure + access control bypass
- SSRF → internal services
- Low-severity → high-impact path
- Multi-step paths crossing component boundaries (user → admin, external → internal, read → write, single-tenant → cross-tenant)
- Continue until maximum privilege / data exposure / control is reached

**Phase 5 — Persistent Testing**
When initial attempts fail: research technology-specific bypasses, try 10+ alternative techniques, test edge cases, test with different client contexts, revisit with new information, try timing-based and blind exploitation.

**Phase 6 — Comprehensive Reporting**

**Agent strategy (deep mode)**:
After recon, decompose hierarchically:
1. Component level (Auth System, Payment Gateway, User Profile, Admin Panel)
2. Feature level (Login Form, Registration API, Password Reset)
3. Vulnerability level (SQLi Agent, XSS Agent, Auth Bypass Agent)

Scale horizontally to maximum parallelization. Each agent focuses on one vulnerability type in one specific area.

---

## 3. Target Type Matrix

| Target | Classification | Approach |
|--------|---------------|---------|
| URL / domain | `web_application` | Black-box by default |
| GitHub / git URL | `repository` | Cloned → white-box (static + dynamic) |
| Local directory | `local_code` | Mounted into container → white-box |
| IP address | `ip_address` | Black-box network scan |
| URL + repository | Combined | Source-aware: code informs live testing; live findings focus code review |

**Combined mode (most powerful)**: Static findings drive payload selection and endpoint prioritization for live testing. Dynamic anomalies direct which code paths to review deeper.

---

## 4. Multi-Agent Execution Model

### Roles

**Root Agent** (`strix`) — Orchestrator only. Does NOT perform direct testing.
- Builds target map
- Decomposes into discrete, parallelizable tasks
- Spawns/monitors specialist subagents
- Aggregates findings
- Calls `finish_scan` with executive summary

**Specialist Subagent** — Executor. Has ONE specific task.
- Named for its specialty (`XSS Discovery Agent`, `SQLi Validation Agent`)
- Gets 1–3 related skills (max 5)
- Calls `agent_finish` when done; sends completion report to parent inbox

### Agent Tree Topology

```
Root (strix)
  ├─ Recon Agent (subfinder, nmap, katana, httpx)
  ├─ SQLi Discovery Agent (sql_injection skill)
  │     └─ SQLi Validation Agent (sql_injection skill)
  │           └─ SQLi Reporting Agent (sql_injection skill)
  │                 └─ SQLi Fixing Agent (whitebox only)
  ├─ XSS Discovery Agent (xss skill)
  │     └─ XSS Validation Agent
  │           └─ XSS Reporting Agent
  ├─ Auth Specialist (authentication_jwt + business_logic skills)
  └─ IDOR Agent (idor skill)
        └─ IDOR Validation Agent
              └─ IDOR Reporting Agent
```

Agents are created **reactively** as discoveries are made — not all pre-planned at scan start.

### Four-Agent Vulnerability Chain (White-box)

```
1. Discovery Agent   → finds candidate vulnerability (static or dynamic signal)
2. Validation Agent  → proves it's real with a working PoC
3. Reporting Agent   → calls create_vulnerability_report (ONLY reporting agents file vulns)
4. Fixing Agent      → patches code + verifies fix removes the vulnerability
```

### Three-Agent Vulnerability Chain (Black-box)

```
1. Discovery Agent  → finds candidate
2. Validation Agent → proves PoC works
3. Reporting Agent  → files report (NO fixing in black-box)
```

### Spawning Rules

- `view_agent_graph` BEFORE spawning — confirm no existing agent already covers this scope
- Max 5 skills per agent; prefer 1–3 related skills
- Do NOT spawn for trivial one-shot probes — do it yourself
- Each agent has ONE non-overlapping task
- Only Reporting Agents call `create_vulnerability_report`
- Children inherit parent conversation history as background context

### Agent Communication

- Parent spawns child → continues immediately in parallel
- Child completion → `agent_finish` → completion report lands in parent's inbox
- Parent waits → `wait_for_message` (timeout 600s)
- Urgent inter-agent messages → `send_message_to_agent`
- Stop misbehaving agent → `stop_agent` (graceful, current turn finishes)

---

## 5. White-Box Static Analysis Pipeline

Executed before dynamic testing when source is available. The 4-tool coverage baseline:

| Tool | Purpose | Output |
|------|---------|--------|
| `semgrep` | Security-first fast triage, custom patterns | `/workspace/.strix-source-aware/semgrep.json` |
| `ast-grep (sg)` | Structural code patterns, route/sink mapping | `/workspace/.strix-source-aware/ast-grep.json` |
| `gitleaks` + `trufflehog` | Secret detection (working tree + git history) | separate JSON files |
| `trivy fs` | Dependency vulns, misconfigs, licenses | `/workspace/.strix-source-aware/trivy-fs.json` |

**AST target derivation**: Build `sg-targets.txt` from `semgrep.json` scope first (`paths.scanned` → fallback to unique `results[].path`). Run `xargs sg run` on that list. Only use path-heuristic fallback if semgrep scope is unavailable.

**Key invariant**: Static findings are hypotheses. Dynamic PoC validation is required before any report is filed. SAST output drives prioritization, not conclusions.

### Additional JavaScript coverage
- `retire` (vulnerable library detection)
- `eslint` (no-eval, no-implied-eval)
- `jshint` for bundles after `js-beautify` deobfuscation
- `JS-Snooper` / `jsniper.sh` for endpoint mining in minified bundles

---

## 6. Dynamic Testing Toolkit (Sandbox)

All testing executes inside a Kali Linux Docker container. All HTTP/S traffic routes through a Caido proxy sidecar on port 48080 (visible to agents via proxy tools).

| Category | Tools |
|----------|-------|
| Recon | `subfinder`, `nmap`, `ncat`, `httpx`, `naabu` |
| Crawling | `gospider`, `katana` |
| Fuzzing | `ffuf`, `dirsearch`, `arjun` (parameter discovery) |
| Vuln scanning | `nuclei`, `sqlmap`, `zaproxy`, `wapiti` |
| Code analysis | `semgrep`, `ast-grep`, `tree-sitter`, `bandit`, `trufflehog`, `gitleaks`, `trivy` |
| Auth testing | `jwt_tool` |
| WAF detection | `wafw00f` |
| OOB testing | `interactsh-client` (generates `*.oast.fun` callbacks) |
| Proxy | Caido CLI (all traffic captured, queryable via proxy tools) |
| Browser | Playwright agent-browser (CDP-based, separate from proxy) |
| Languages | Python 3, Go, Node.js; install additional tools via `apt`/`pip`/`npm`/`go install` |

**Caido integration**: Agents query captured proxy history via `list_requests`, `view_request`, `repeat_request`. Python scripts inside the container import `caido_api` directly for automation.

**Payload spraying rule**: For trial-heavy vectors (SQLi, XSS, XXE, SSRF, RCE, auth, deserialization) — NEVER iterate payloads manually. Always spray via Python scripts through `exec_command` or terminal tools. Use `web_search` to fetch latest bypass sets.

---

## 7. Vulnerability-Specific Methodology

Each vulnerability has a prescribed methodology: attack surface identification → reconnaissance → key vulnerability patterns → testing order → validation requirements → false positive checklist.

### IDOR / BOLA

**Attack surface**: Every object reference across all transports (REST, GraphQL, WebSocket, gRPC, mobile).

**Recon approach**:
- Use list/search/export endpoints as ID seeders
- `arjun` to discover hidden parameters (`?include_deleted=1`, `?as_user=`, `?owner_id=`)
- Map Subject × Object × Action matrix

**Key patterns**: horizontal (same type, different user), vertical (lower priv → admin), cross-tenant (org boundary), secondary IDOR (extract IDs from notifications/emails/webhooks then use them), batch operations (validate first element only), job IDs.

**Bypass techniques**: content-type switching, JSON duplicate keys, parameter pollution (`id=1&id=2`), case/alias alternation, cache key confusion.

**Validation requirements**:
1. Demonstrate access to an object not owned by the caller
2. Show the same request fails with proper auth
3. Prove cross-channel consistency (at least 2 transports)
4. Provide request/response evidence for both owner and non-owner

---

### SQL Injection

**Detection hierarchy**: error-based → boolean-based → time-based → OOB (prefer quietest oracle first).

**Attack surface emphasis**: ORM raw query APIs (`whereRaw`, `orderByRaw`), ORDER BY / GROUP BY / LIMIT / OFFSET, JSON/JSONB operators, full-text search helpers, stored procedures, batch/bulk endpoints, report generators.

**DBMS-specific**: MySQL (extractvalue, JSON functions, SLEEP), PostgreSQL (pg_sleep, JSONB operators, COPY), MSSQL (xp_dirtree for OOB, WAITFOR DELAY), Oracle (UTL_HTTP, dbms_lock.sleep).

**Write primitives**: auth bypass via tautologies, privilege changes (UPDATE role/feature flags), file write (INTO OUTFILE, COPY TO), job/proc creation.

**Bypass techniques**: whitespace (/**/, ideographic space), keyword splitting, encoding chains, subselect/CTE/lateral join to hide payload shape.

**Validation requirements**:
1. Reliable oracle (toggleable predicate)
2. Verifiable metadata extraction (version, current user, DB name)
3. Non-trivial target access or modification
4. Reproducible requests differing only in injected fragment

---

### XSS

**Context classification first** — type of XSS matters less than context: HTML node, attribute, URL, JS string, event handler, CSS, SVG.

**DOM XSS priority sources**: `location.*`, `document.referrer`, `postMessage`, storage, service worker messages.
**DOM XSS priority sinks**: `innerHTML`/`outerHTML`/`insertAdjacentHTML`, `eval`, `setTimeout` with strings, `document.write`.

**CSP bypass techniques**: JSONP endpoints, script gadgets (function constructors), base tag injection to retarget relative script URLs, import maps, `data:`/`blob:` if allowed.

**Framework-specific**: React (`dangerouslySetInnerHTML`), Vue (`v-html`), Angular (legacy expression injection), Svelte (`{@html}`).

**Validation requirements**:
1. Minimal payload + context (sink type) + before/after DOM/network evidence
2. Cross-browser execution or parser-specific explanation
3. Bypass proof of stated defenses (sanitizer, CSP)
4. Impact beyond `alert()` — data exfiltrated, action performed, persistence achieved

---

### SSRF

**Oracle hierarchy**: OAST DNS/HTTP callbacks first (`interactsh-client -v` in sandbox), then internal address variants, then protocol variations.

**High-value targets (priority order)**:
1. AWS IMDSv1: `http://169.254.169.254/latest/meta-data/iam/security-credentials/<role>`
2. AWS IMDSv2: PUT token then GET with `X-aws-ec2-metadata-token` header
3. GCP: `http://metadata.google.internal/computeMetadata/v1/` + `Metadata-Flavor: Google`
4. Azure: `http://169.254.169.254/metadata/instance` + `Metadata: true`
5. Kubernetes API, kubelet (10250/10255), internal services (Redis 6379, Docker 2375, Elasticsearch 9200)

**Protocol exploitation**: `gopher://` for raw Redis/SMTP/FastCGI, `file://` for local files, `dict://` for port probing.

**Bypass techniques**: IP encoding variants (decimal, hex, octal, IPv6), DNS rebinding (short TTL), URL parser differentials, redirect chains (allowlist at initial URL, redirect to internal), CRLF injection into headers.

**Blind SSRF**: Use `interactsh-client` to confirm egress. Map internal ports by diffing timing/response-size/TLS-error-class.

**Validation requirements**:
1. Prove outbound server-initiated request (OAST hit or internal-only response diff)
2. Access non-public resource (metadata, internal admin, service port)
3. Demonstrate minimal-impact credential access or harmless internal read
4. Confirm reproducibility and document what controls scheme/host/headers/method

---

### Authentication / JWT

**Quick wins first**: RS256→HS256 confusion, `"alg":"none"` acceptance — if algorithm pinning is unclear. Otherwise focus on header key control.

**Header manipulation attacks**: `kid` injection (path traversal, SQL injection in key lookup), `jku`/`x5u` abuse (host attacker JWKS), `jwk` header injection (embed attacker key inline).

**Claims validation gaps**: `iss`/`aud` not enforced (cross-service reuse), `scope`/`roles` trusted from token without re-derivation, `exp` not enforced, `typ`/`cty` not enforced (ID token accepted as access token).

**Token confusion**: swap access token for ID token across services that only check signature.

**JWT Tool**: `jwt_tool -t <url> -rh "Authorization: Bearer <token>" -M at` runs full attack matrix; `jwt_tool <token> -C -d <wordlist>` brute-forces HMAC secrets.

**Validation requirements**:
1. Forged or cross-context token acceptance proof
2. Access token vs ID token confusion at an API
3. Refresh token reuse without rotation detection
4. Header abuse (kid/jku/x5u/jwk) leading to attacker-controlled key selection

---

### RCE

**Oracle hierarchy**: OAST (DNS/HTTP via `interactsh-client`) → timing (short gated delays, NOT long sleeps) → direct output.

**Command injection delimiters**: Unix (`;`, `|`, `||`, `&`, `&&`, backtick, `$()`, `${IFS}`, newline), Windows (`&`, `|`, `||`, `^`). Argument injection into CLI flags is often easier than separator injection.

**Template injection quick probes**: `{{7*7}}` (Jinja2/Twig), `${7*7}` (Freemarker), `<%= %>` (EJS). If confirmed → escalate to exec payload.

**Deserialization targets**: Java gadget chains (ysoserial), PHP unserialize/PHAR, Python pickle/yaml.load, .NET BinaryFormatter.

**Media pipelines as sinks**: ImageMagick, Ghostscript, ExifTool, LaTeX `\write18`, ffmpeg.

**Evasion**: `${IFS}` whitespace, token splitting (`w'h'o'a'm'i`), variable building, base64 stagers.

**Post-RCE container checks**: `/.dockerenv`, `capsh --print`, docker.sock presence, Kubernetes service account token at `/var/run/secrets/kubernetes.io/serviceaccount`.

**Validation requirements**:
1. Minimal reliable oracle (DNS/HTTP/timing) proving code execution
2. Command context: uid, gid, cwd, env
3. Persistence or controlled file write
4. Container boundary crossing attempts and results

---

### Business Logic

**Invariant model first**: conservation of value (ledger balance), uniqueness (idempotency), monotonicity (non-decreasing counters), exclusivity (one active subscription).

**Key attack patterns**: state machine step skipping (call API endpoints out of order), concurrency races (parallel identical operations to bypass atomic checks), idempotency key scope abuse (reuse other users' keys), numeric tricks (negative amounts, zero-price, cross-currency arbitrage, rounding exploitation).

**Quota/limit bypass**: off-by-one on time resets, distributed counter inconsistency, reservation/hold leaks.

**Role/feature gate abuse**: feature flags enforced client-side only, role transitions leaving stale capabilities.

**Validation requirements**:
1. Invariant violation with side-by-side evidence (intended vs abused flow)
2. Durable observable state change (ledger entry, admin view, email)
3. Quantified impact per action and at scale

---

### SSTI

**Fingerprinting approach**: inject `{{7*7}}`, `${7*7}`, `<%= 7*7 %>` to identify engine. Observe response for `49` (Jinja2/Twig/Freemarker/EJS) or literal string (safe template).

**Escalation after fingerprint**:
- Jinja2: `{{cycler.__init__.__globals__['os'].popen('id').read()}}`
- Twig: `{{_self.env.registerUndefinedFilterCallback('system')}}{{_self.env.getFilter('id')}}`
- Freemarker: `<#assign ex="freemarker.template.utility.Execute"?new()>${ ex("id") }`
- EJS: `<%= global.process.mainModule.require('child_process').execSync('id') %>`

---

### Race Conditions

Testing approach: parallelize identical operations using Python `asyncio`/`aiohttp` or `ffuf` concurrent mode. Target: create/apply/redeem/transfer endpoints. Prove TOCTOU by showing duplicate side-effects before state updates.

---

## 8. Validation Contract

**The reporting contract is strict**. Every `create_vulnerability_report` call must include:

| Field | Requirement |
|-------|------------|
| `title` | Specific (not generic class) — e.g., "SQL Injection in /api/login username parameter" |
| `description` | How discovered + what it is |
| `impact` | Business risk + data at risk |
| `target` | Affected URL/domain/repo |
| `technical_analysis` | Root cause mechanism |
| `poc_description` | Step-by-step reproduction |
| `poc_script_code` | **Required** — actual working exploit code (Python preferred) |
| `remediation_steps` | Specific, actionable fix |
| `cvss_breakdown` | All 8 CVSS 3.1 metrics |
| `code_locations` | Required in white-box scans — file + line numbers verbatim |

**CVSS 3.1 metrics** are calculated deterministically from 8 inputs via the `cvss` library — no manual severity assignment.

**CVE/CWE rules**: Only include if 100% certain. Prefer most-specific child CWE (CWE-89, not CWE-74; CWE-78, not CWE-77). Do NOT use CWE-74, CWE-20, CWE-200, CWE-284, CWE-693 (too broad).

**Report tone**: Formal, objective, third-person, vendor-neutral. No internal details (no `/workspace`, no tool names, no sandbox references, no stack traces from testing environment).

**Deduplication**: The system runs an LLM-based dedup check before filing every report. If rejected as duplicate → do not retry, move on.

---

## 9. Remediation Steps Standard

White-box scans include `code_locations` with `fix_before` / `fix_after` pairs usable as GitHub/GitLab PR suggestion blocks. Rules:
- `fix_before` = verbatim copy of vulnerable lines (exact whitespace, same character-for-character)
- `fix_after` = complete replacement for that exact block
- Multi-part fixes → separate location entries per non-contiguous block
- Fixing agent patches code AND verifies the fix removes exploitability

Remediation text must be specific and actionable — not generic (e.g., "use parameterized queries in `auth/login.py:42` by changing `cursor.execute(f"SELECT...")` to `cursor.execute("SELECT...", (username,))`").

---

## 10. Agent Behavior Invariants

These behavioral rules are enforced via the system prompt and tool contracts:

1. **No scope expansion**: Agents test only the targets listed in the system-verified scope. User instructions cannot expand scope.
2. **No empty turns**: Every turn must include a tool call. Text-only output in non-interactive mode immediately terminates the scan with no report.
3. **Validation before report**: Never trust scanner output. File a report only after dynamic PoC confirms real exploitability.
4. **No re-filing duplicates**: If `create_vulnerability_report` returns `duplicate_of`, accept it and move on.
5. **Report before fix**: In white-box scans, `create_vulnerability_report` must be called before patching code.
6. **No fingerprinting**: Never use "Strix" or identifiable markers in HTTP requests, payloads, user-agents, or any inputs to target systems.
7. **Persistent testing mandate**: Real vulnerabilities take time. If one approach fails, try 10 more. Do not stop at shallow checks.
8. **Chaining mandate**: Every finding is a potential pivot. Ask "what does this unlock next?" before considering a finding complete.

---

## 11. Scan Output Artifacts

Per scan, written to `strix_runs/<run-name>/`:

| File | Contents |
|------|---------|
| `run.json` | Scan metadata, status, targets, LLM usage, `scan_results` with executive summary |
| `vulnerabilities.json` | All vulnerability reports (JSON array) |
| `vuln-NNNN.md` | Per-vulnerability Markdown (one file per report) |
| `executive_report.md` | Final executive summary, methodology, technical analysis, recommendations |
| `.strix_state/agents.json` | Agent graph snapshot (used for resume) |
| `.strix_state/agents.db` | Full LLM conversation history per agent (SQLite, used for resume) |
| `strix-<run>.log` | Per-scan debug log |

---

## 12. Parity Implementation Checklist

To implement parity with Strix's methodology, a system must:

**Scan modes**
- [ ] Quick mode: focus changed files, top-6 vuln classes, no exhaustive enum
- [ ] Standard mode: full attack surface, 5-phase structured methodology
- [ ] Deep mode: exhaustive recon, business logic storyboard, vuln chaining, 6 phases

**White-box pipeline**
- [ ] 4-tool static baseline: semgrep + ast-grep + gitleaks/trufflehog + trivy fs
- [ ] AST target list derived from semgrep scope (not path heuristics)
- [ ] Static findings treated as hypotheses; dynamic validation required

**Multi-agent system**
- [ ] Root agent as coordinator only (no direct testing)
- [ ] 3-agent chain (black-box): discovery → validation → reporting
- [ ] 4-agent chain (white-box): discovery → validation → reporting → fixing
- [ ] Reactive spawning (agents created as discoveries are made)
- [ ] Only reporting agents file vulnerability reports
- [ ] Agent graph visibility (equivalent of `view_agent_graph`)

**Vulnerability coverage**
- [ ] All 21 vulnerability skills: idor, sql_injection, xss, ssrf, authentication_jwt, rce, ssti, business_logic, csrf, xxe, race_conditions, path_traversal_lfi_rfi, insecure_file_uploads, mass_assignment, nosql_injection, header_injection, http_request_smuggling, subdomain_takeover, open_redirect, information_disclosure, broken_function_level_authorization

**Validation contract**
- [ ] PoC script required for every report
- [ ] CVSS 3.1 calculated from 8-metric breakdown
- [ ] LLM-based or semantic deduplication before filing
- [ ] Code locations with verbatim fix_before/fix_after for white-box findings

**Report quality**
- [ ] Formal tone, no internal details
- [ ] Specific remediation (file + line + code change)
- [ ] Report before fix in white-box workflows

---

## Known Gaps / Uncertainties

- `reconnaissance/` skill files (subfinder, nmap, etc.) not fully read — recon tool usage details may have additional methodology
- `frameworks/` skill content (fastapi, nestjs, nextjs) not fully analyzed — framework-specific attack surfaces may have additional patterns
- `protocols/graphql.md` not read — GraphQL-specific testing methodology beyond what appears in IDOR/SSRF skills
- `cloud/kubernetes.md` not read — full Kubernetes attack surface methodology
- `technologies/supabase.md`, `firebase_firestore.md` not read — BaaS-specific patterns
- Whether `standard.md` differs materially from the combination of `deep.md` + time constraint was not deeply analyzed
