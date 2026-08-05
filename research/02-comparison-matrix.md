# 02 — Comparison Matrix (Pass C)

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commits in [00-scope](00-scope.md).*

Cell vocabulary: `native` / `via-ext` (via-extension) / `partial` / `external` / `absent` / `unclear`. Notes given wherever a label alone would mislead. Citations live in dossiers 03–08 and ledger 11; cells state the dossier section when non-obvious. Columns: **OC**=opencode · **GS**=goose · **OH**=OpenHands(SDK+Canvas) · **PI**=pi-mono · **MSA**=mini-swe-agent · **CL**=cline.

## 1. Product & interface

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| TUI | native | native (REPL + npm TUI) | external (CLI in 3rd repo) | native | partial (rich REPL; textual inspector only) | native (OpenTUI) |
| IDE integration | native ×2 (VS Code ext, ACP/Zed) | via-ACP | partial (VS Code service in agent-server, no plugin) | absent (RPC is the path) | absent | native (VS Code) + external (JetBrains, legacy bridge) |
| Desktop/web UI | native (Electron, web, console) | native (Electron) | native (the app repo IS this) | absent | absent | partial (hub dashboard; Tauri example) |
| Headless/CI | native (`run --auto --format json`, GH Action) | native (`goose run --no-session`) | native (Python API; GH workflow examples) | native (`-p`/`--mode json`) | native (batch runner) | native (CLI yolo/JSON; evals drive it) |
| Server API | native (Hono HTTP+SSE, OpenAPI) | native (ACP over HTTP+WS — no REST) | native (FastAPI REST+WS, ~20 routers) | partial (experimental CBOR server) | absent | native (hub WS daemon, token-authed) |
| SDK | native (generated JS SDK; in-process server) | native (uniffi Py/Kotlin + TS, all ACP clients) | native (the SDK repo; typed pydantic) | native (its pitch; injectable loop) | partial (Protocol duck-typing, forkable) | native (published npm packages, pre-1.0) |
| Multi-session | native | native | native | partial (per-process; concurrent-cwd by convention) | absent | native (hub attach/detach) |
| Session branching/forking | native (fork by messageID) | native (`--resume --fork`, `--edit`) | native (event tree, parent_id) | native (JSONL tree, in-place branch) | absent | partial (checkpoint restore + edit-regenerate; no tree) |
| Approval UX | native (once/always/reject + feedback) | native (ActionRequired + channel) | native (risk-conditional policies) | absent by philosophy (via-ext) | partial (InteractiveAgent only; batch path none) | native (per-tool policy + category toggles) |
| Diff/checkpoint UX | native (git snapshots + revert) | partial (edit previews; snapshots dir unverified) | partial (git module/router; no auto per-step checkpoint) | absent (example extensions) | absent | native (real-git refs; destructive restore w/ stash txn) |

## 2. Model layer

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Provider abstraction | native (AI SDK v6 + models.dev) | native (Provider trait) | native (LiteLLM, pinned 1.93.0) | native (pi-ai per-API modules) | native (litellm + 3 alt gateways) | native (gateway over Vercel AI SDK) |
| Where it leaks | family-keyed prompts+toolsets; 16 patched deps; GitLab bridge | thinking-block surgery in core loop; toolshim | litellm field names/exceptions/pin | declared `compat` flags (leaks made explicit) | model_kwargs passthrough (thin by design) | vendors/+routing/ branches (contained) |
| OpenAI-compatible + local | native (config baseURL) | native (env/declaratives + candle local) | native (base_url, ollama) | native (models.json; llama.cpp) | native (api_base kwargs) | native (openai-compatible vendor) |
| Model-specific adaptations | extensive (prompt+toolset fork on model ID) | native (surgery + toolshim) | native (model_features registry + per-model presets) | native (per-API modules, compat flags) | minimal (anthropic cache flag only) | native (vendors, routing, middleware) |
| Reasoning-model support | native | native | native (responses API, thinking blocks) | native (thinking levels/budgets) | partial (passthrough) | native |
| Prompt caching | native (breakpoints, prompt_cache_key) | native (anthropic ephemeral + stable tool order) | native (2-tier system prompt for cross-conversation hits) | native (anthropic + prompt_cache_key) | partial (anthropic-only auto flag) | native (cache_control routing; med confidence) |
| Token/cost accounting | native (Decimal, cache-aware, tiering) | native (usage_ledger table) | native (ConversationStats + budget enforcement) | native (per-message cost in session) | native (cost only, no token field) | native (incl. BYOK upstream cost) |
| Retry/fallback | retry native; no model fallback | retry native; recipe retry gates | retry + FallbackStrategy (alternate LLM list) | retry native; fallback absent (UNVERIFIED low) | retry native (tenacity) | partial (empty-response middleware; no chain) |
| Can drive another complete agent | partial (be-driven yes; drive = MCP only) | **native (ACP client: 5+ agents as providers)** | native (ACPAgent engine swap + delegate) | via-ext (subagent example via RPC) | absent | partial (is ACP agent; cannot drive) |

## 3. Tool system

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Built-ins | 17 tools | platform exts (developer/analyze/todo/summon/…) | terminal/file_editor/apply_patch/browser/glob/grep/tracker/delegate | 7 tools | 1 (bash) | 9 SDK tools + spawn_agent/team |
| Registration/schemas | Effect Schema→JSON; plugin rewrite hook | rmcp Tool JSONSchema + normalize | pydantic action_type→OpenAI schema | TypeBox | single hardcoded schema | Zod→JSON Schema |
| MCP client (spec rev) | native, SDK 1.29.0 → **2025-11-25** (external); roots only, sampling/elicitation off | native, rmcp 3.0.0 → **2025-11-25** (external) | native (fastmcp≥3.0.0; rev delegated) | absent by philosophy | absent | native ×2: core stdio-only **2024-11-05**; ext official SDK all transports |
| MCP server | absent | native (bundled servers) | partial (mcp_router/OAuth in agent-server) | absent | absent | absent |
| ACP (role, version) | agent role, v1 (sdk 0.21.0) | **both roles** (crate 1.0) | client role (ACPAgent) + Canvas drives any ACP agent | absent | absent | agent role (CLI --acp) |
| LSP | native (diagnostics fed back post-edit) | absent | absent | absent | absent | absent |
| Browser/web tools | webfetch + websearch (3rd-party A/B endpoints) | via-ext (web_scrape in MCP server) | native (browser-use wrapper) | absent (bash+curl) | absent | partial/legacy (puppeteer in ext; not in SDK toolset — UNVERIFIED) |
| Shell execution | host, tree-sitter-derived ask patterns | host login shell (flatpak escape) | persistent tmux/PTY (host or in-container) | host detached spawn, tree-kill | fresh subprocess per action (stateless) | node spawn + kill-tree; VS Code terminal-integrated |
| File-edit mechanism | 9-stage fuzzy cascade + apply_patch (model-gated) | exact match, "did you mean" recovery (no fuzzy) | exact str_replace, strip-retry, no fuzzy | exact + unicode-normalize fallback | bash only (sed/heredoc) | exact unique match, no fuzzy; apply_patch shipped but disabled in presets |
| Git integration | native (snapshots, revert, pr, worktrees) | partial (shell only; security patterns tested) | native (git module + router + PR prompts) | absent (bash) | absent (bash; template teaches git diff) | native (checkpoint refs, worktree, commit-msg gen) |
| Custom-tool API | plugin tool() | any MCP server / extension type | ToolDefinition subclass | pi.registerTool (TypeBox) | fork/duck-type | AgentTool + plugins + host executors |
| Plugin/extension lifecycle | native; many hooks `experimental.*` | native (+ OSV malware check of stdio cmds) | native (skills/hooks/subagents md) | native (trust-gated dirs, hot reload) | absent | native (plugins, hooks; pre-1.0 churn) |
| Tool-result truncation | limits + spill-to-file | >200k → spill-to-file pointer | serialization-time char limit | 2k lines/50KB + spill path | fixed 5k head + 5k tail per obs | 48k caps, head/tail elision |

## 4. Context engineering

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Repo maps | absent | partial (tree-sitter analyze on demand) | absent | absent | absent | absent |
| Search/retrieval | ripgrep glob/grep; no index | shell + chatrecall | glob+grep tools | ripgrep w/ JS fallback | bash | ripgrep + regex fallback |
| Explicit file selection | native (file parts read server-side) | via hints/shell | native (mentions in Canvas; skills) | via prompt | via prompt | native (@-mentions) |
| LSP-derived context | native | absent | absent | absent | absent | absent |
| Skills (SKILL.md) | native (+Claude-dir compat) | native (agentskills.io, ~/.agents/skills) | native (+triggers, marketplace) | native (+Claude/Codex dirs) | absent | native (dynamic skills tool) |
| AGENTS.md equivalent | native (+CLAUDE.md, nested lazy, remote URLs) | native (+.goosehints, nested, global) | native (as always-active skill) | native (+CLAUDE.md, ancestors, worktree shadow) | absent | native (+.clinerules legacy) |
| Memory | absent | via-ext (memory MCP server) | native (2-tier MEMORY.md, 6k budget) | absent | absent | absent (memory bank gone) |
| Session history | SQLite | SQLite (+usage ledger) | event-file tree | JSONL tree | JSON trajectory | SQLite manifests + atomic files |
| Compaction: preserved vs lost | summary + last ~2 turns; DB rows kept; tool outputs pruned to 2k chars | summary; originals kept as invisible rows; model told to hide compaction | keep_first 2 + summary + tail; forgotten span = 1 summary | summary + last ~20k tokens; verbatim old messages lost from context | **absent** (per-obs truncation only) | agentic summary + preserveRecentTokens; canonical transcript fully preserved |
| Context isolation | subagent child sessions | subagent sessions; recipes | subagent LocalConversations | separate pi processes (example) | absent | spawn_agent subagent mode |
| Durable external state | files + SQLite + git snapshots | files + SQLite | files + event log + MEMORY.md | files + JSONL sessions | trajectory JSON only | files + SQLite + git refs + cron specs |
| Large-repo/long-task behavior | auto-compact + auto-continue (drift risk) | 3 compaction mechanisms + turn budget | condenser @240 events + stuck detector | compaction + tree branch summarization | hits context limit or cost cap; no degradation path | compact-once-retry then terminal |

## 5. Planning & agency

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Plan vs build modes | native (plan agent edit-denied via permissions) | partial (chat mode; plan.md prompt) | native (planning preset + planning_file_editor) | absent by philosophy | absent | native-but-prompt-enforced for shell (plan keeps bash enabled) |
| Todos | native (todowrite) | native (todo ext) | native (task_tracker) | absent by philosophy ("confuse models") | absent | native (focus-chain md files) |
| Subagents real or prompts? | **real** (child sessions, resumable, background) | **real** (own Agent + session + recipe) | **real** (sub-LocalConversations; md frontmatter defs) | via-ext (real child processes in example) | absent | **real** (spawn_agent, subagent session mode) |
| Delegation | task tool | summon/orchestrator + sub_recipes | delegate tool (spawn/delegate) | example ext | absent | spawn_agent + teams |
| Parallelism | parallel tools + background tasks | select_all tool futures | ParallelToolExecutor | parallel tools default | batch-level only | parallel toolExecution option |
| Workflow graphs | absent | partial (recipes + sub_recipes) | partial (workflow tool pkg) | absent | absent | partial (cron/event automation specs) |
| Recipes/declarative tasks | markdown agents/commands | **native** (YAML recipes: params, response schema, retry gates) | via skills/subagent md | prompt templates | YAML config IS the task | workflows + cron md specs |
| Background execution | native (background tasks/jobs) | partial (scheduler; steering) | native (automation pkg + bash service) | absent by philosophy (tmux) | absent | native (cron subsystem, detached hub, bg terminal) |
| Scheduling | absent | **native** (cron in-process) | via-ext (openhands-automation) | absent | absent | native (cron/event md specs) |
| Stopping conditions | idle finish + doom-loop guard | max_turns 1000 + stop-hook veto + grind | max_iteration 500 + budget + stuck detector + critic | queues empty + shouldStopAfterTurn (SDK) | step/cost/wall-time limits + sentinel | maxIterations + submit_and_exit + completion nudges |
| Interrupted-run recovery | sessions resumable; orphaned parts marked | resume/fork; steering | event log resume; pause/resume | resume session; `--fork` | trajectory saved every step; no resume verified | session resume; checkpoint restore |

## 6. Execution environment

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Host exec | default | default | default (local) | only mode | LocalEnvironment | default |
| Containers | absent | opt-in (extensions in Docker) | **native** (whole engine in container = the sandbox) | external (documented patterns: Gondolin/Docker) | **native** (docker/singularity per task) | absent |
| Sandbox technology | none | Docker opt-in; flatpak escaped | Docker/Apptainer/sysbox-runc (remote) | none (user's OS boundary) | docker, singularity, **bubblewrap**, contree, Modal | none |
| Remote workers | attach to remote server | ACP remote serve | **native** (agent-server per workspace, port-ranged) | experimental CBOR server | swerex_modal (cloud) | hub RemoteRuntimeHost |
| Kubernetes | absent (infra pkg = hosted product) | absent | native (helm chart for Canvas; k8s deploy) | absent | absent | absent |
| FS/network isolation | absent | absent (inspection only) | only via container/remote choice | absent | via sandbox env choice | absent |
| Credential handling | auth store + env; MCP OAuth | **keyring** + fallback | SecretRegistry + redaction; OAuth lifecycles | 0o600 auth.json + OAuth refresh | env/.env | CLI keys, VS Code secrets, hub token perms |
| Reproducibility | models.dev fetch at runtime (variance) | declaratives compiled in | litellm pin + image pins; uvx-at-launch variance | model catalog is build artifact | YAML + pinned datasets (best) | generated catalog; hub auto-spawn |
| Multi-user | hosted console (commercial) | absent | partial (flat session API keys; cloud = product) | absent | absent | partial (hub tokens, single-user daemon) |

## 7. Security & authority

Key question — **is each boundary OS/runtime-enforced or a UI prompt the model could route around?**

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Boundary type | in-process rules + prompt (no OS) | in-process prompt (+LLM judge) | prompt-policy; OS boundary **only if** container workspace chosen | none (external OS) | none in batch; sandbox env = real OS boundary when chosen | in-process policy + prompt (no OS) |
| Allow/deny/ask rules | native (wildcard, last-match-wins, per-agent) | native (per tool + readonly annotations) | native (policies over risk levels) | absent | whitelist regex (interactive only) | native (ToolPolicy + wildcard presets) |
| Command-pattern matching | native (tree-sitter arity prefixes) | partial (git security patterns) | absent (risk-level based) | absent | whitelist regex | partial (category toggles) |
| Path restrictions | partial (external_directory ask) | absent | absent (workspace = container scope) | absent | cwd only | absent |
| Network restrictions | absent | absent (egress inspection, no blocking) | only via container | absent | via sandbox | absent |
| Prompt-injection defenses | partial (permission gates, doom-loop) | native-heuristic (scanner + LLM inspectors; no blocking) | partial (model-self-reported risk default; GraySwan optional) | absent, documented as unpreventable | absent | absent |
| Auditability | SQLite + snapshots + OTel opt-in | SQLite + telemetry (3 visibility projections caveat) | full event log (canonical) | JSONL sessions (published-dataset-grade) | trajectory JSON | canonical transcript + checkpoint refs |
| Reversibility | native (snapshot revert) | partial | partial (git) | via git (user) | via git (user) | native (checkpoint restore — itself destructive w/ txn) |
| Default trust posture | ask (default rules) | approve mode default (smart = LLM) | confirmation off unless analyzer+policy set | full trust, documented | interactive: confirm; batch: full-auto | ext: category approvals; **SDK default auto-approves unlisted tools** |

## 8. Verification

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Test/lint/typecheck execution | shell + auto-formatters + LSP diagnostics | shell | shell + optional linting in editor | shell | bash | shell + terminal integration |
| Validation loops | LSP feedback post-edit | recipe retry success-checks (shell gates) | critic + iterative refinement + Stop hooks | exit codes only | none (model self-policed) | YOLO prompt mandates test-until-pass; `verified` flag on submit |
| Independent review | absent | **native** (`goose review`, .agents/checks/*.md subagents) | via critic/hooks config | via-ext (reviewer example) | absent | via hooks |
| Acceptance-criteria tracking | todos | recipe response json_schema | task_tracker + critic | files | none | focus-chain |
| Evidence capture | patch parts + snapshots | session ledger | event log | JSONL + cost | trajectory JSON | transcript + checkpoints |
| Benchmark integration | absent | evals/ (harbor) | integration/behavior tests w/ LLM judge; **no SWE-bench harness in-repo** (77.6 badge → external sheet) | evals pkg (harness-dev evals) | **native** (SWE-bench official runner) | evals/ + cline-bench vs CLI |
| Exit-0 vs requirement-demonstrated | not distinguished | partial (success-check cmds are still exit-0 checks) | partial (critic is LLM judgment) | not distinguished | not distinguished | partial (`verified` flag is model-asserted) |

**No subject mechanically distinguishes "command exited 0" from "requirement demonstrated"** — the closest approximations are goose's recipe success-checks (operator-authored commands), OpenHands' critic (LLM), and cline's model-asserted `verified` flag. This is a study-level absence finding.

## 9. Extensibility & portability

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Plugin API stability | experimental-prefixed hooks | extensions stable-ish (MCP); ACP `_meta` unstable | pydantic APIs, versioned pkgs | "minor = breaking, no majors" | n/a (fork) | pre-1.0 (0.0.69) |
| Skills | native | native | native | native | absent | native |
| MCP servers usable | yes (client) | yes (both) | yes (client + router) | via-ext only | no | yes (client; weaker headless) |
| ACP endpoints | agent | both | client (+Canvas as hub) | none | none | agent |
| Hooks/events | plugin hooks + SSE | 11 HookEvents (PreToolUse deny, Stop veto) | shell-command hooks + event stream | extension events | none | subprocess hooks + event stream |
| Custom UI/agent/tool/provider/runtime | UI:SDK; tools:plugins; providers:config; runtime:no | all via protocols; runtime:no | all (AgentBase subclass proves loop swap) | all except runtime; loop injectable | fork | UI:hosts; tools/providers:yes; loop:hooks only |
| Config portability | opencode.json + md dirs | YAML + env + keyring | pydantic JSON + md dirs | settings.json + md dirs (very portable) | pure YAML (most portable) | JSON + md dirs |
| Lock-in severity | medium (plugins, SQLite, permission schema) | medium (recipes harness-locked; skills/AGENTS.md portable) | low-medium (standards-heavy; litellm pin) | low (files everywhere; API churn is the cost) | **lowest** | medium (pre-1.0 SDK; file formats portable) |

## 10. Developer experience

| Dimension | OC | GS | OH | PI | MSA | CL |
|---|---|---|---|---|---|---|
| Setup cost | low (binary + env key) | low-med (configure wizard) | med (uvx/Docker for isolation) | low (npm -g) | low (pipx) | low (npm -g) |
| Config clarity | JSON schema'd | YAML + wizard | typed pydantic | documented settings | single YAML | JSON + dirs |
| Code readability | high quality, high complexity (Effect) | monolithic core (4,917-line agent.rs) | giant modules (llm.py 3,197) but typed | high (small core, clean layers) | **highest** (read in a sitting) | mixed (clean SDK; 90-file ext bridge) |
| Abstraction quality | strong services; leaky provider seams | protocol-maximal; loop monolith | strong seams (AgentBase, workspace, condenser) | strongest loop injection points | Protocol duck-typing, minimal | strong layering (shared→llms→agents→core) |
| Testability | 245 test files incl. loop tests | integration + replay fixtures | 665 test files, ~4.8k tests | 409 test files, faux provider | 53 test files | 539 test files across sdk+apps |
| Debuggability | OTel opt-in + SSE events | 3-projection transcripts hurt; otel | event log is ground truth | JSON/RPC event stream | trajectory = messages (best) | typed event stream + hub |
| Add a tool | plugin ~20 lines | MCP server or SKILL.md | ToolDefinition subclass | extension ~20 lines | fork | AgentTool object |
| Add a provider | config entry | declarative YAML | LiteLLM name (zero-code) | models.json entry | litellm string | vendor module + catalog |
| Change the loop | fork | fork (monolith) | subclass AgentBase | SDK hooks (no fork) | edit 190 lines | hooks/prepare-turn seams |
| Embed | serve + SDK / in-process | ACP client / uniffi | pip install + 10 lines | 6-line SDK | import as lib | `new Agent({...})` |
| Ship to a team | strong (server + policy rules) | strong (recipes + keyring + scheduler) | strong (helm/VM + shared server) | weak (DIY guardrails) | n/a | medium (hub + pre-1.0 churn) |
