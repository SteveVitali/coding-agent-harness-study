# 04 — goose (aaif-goose/goose, formerly block/goose)

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commit below.*

Follow-up work (traces not performed in this study): verify rmcp 3.0.0 ProtocolVersion::LATEST spec-date string against crates.io source (resolved externally 2026-08-04 — negotiates 2025-11-25); read acp/server.rs prompt path end-to-end; sample one declarative provider YAML; date the block→aaif-goose repo rename via `gh`.

Pinned commit: `7e431ac6f804fdc5a6fb9262fa2ca5b8b0fd6ce6` (2026-08-04, verified `git rev-parse HEAD`).

## A. Orientation

- **Identity.** "An AI agent" — general-purpose, not code-only (`Cargo.toml:16` description; `README.md:26`). Workspace version **1.45.0**, Rust edition 2021, rust 1.94.1 (`Cargo.toml:9-12`). License **Apache-2.0** (`LICENSE:1-3`, `Cargo.toml:14`). Authors `AAIF <ai-oss-tools@block.xyz>`; repository `github.com/aaif-goose/goose` (`Cargo.toml:13-15`) — the repo has moved off `block/goose`. README states "goose is part of the Agentic AI Foundation (AAIF) at the Linux Foundation" (`README.md:33`). `GOVERNANCE.md:1-13` defines a lightweight maintainer model; the file was added 2025-10-02 (`git log --diff-filter=A -- GOVERNANCE.md`, commit-history). Coupling to Block: authors email + PostHog telemetry module (`crates/goose/src/posthog.rs`), otherwise no hosted-service dependency for core operation.
- **Crates** (`crates/`): `goose` (core agent, ~everything), `goose-cli`, `goose-providers` + `goose-provider-types` (provider impls/trait+wire formats), `goose-mcp` (bundled MCP servers), `goose-sdk` + `goose-sdk-types` (ACP-client SDK, uniffi Python/Kotlin bindings — `crates/goose-sdk/src/lib.rs:1-13`), `goose-local-inference` (candle-based local models; `Cargo.toml:33-35`), `goose-acp-macros`, `goose-download-manager`, `goose-test`, `goose-test-support`.
- **There is no `goose-server` crate anymore.** The API surface is **ACP**: `goose acp` (stdio) and `goose serve` (ACP over HTTP+WebSocket, axum, default port 3284, secret-key auth) (`crates/goose-cli/src/cli.rs:828-890`). The Electron desktop app is itself an ACP client connecting over WebSocket (`ui/desktop/src/main.ts:32,1123,1986`).
- **UI stack** (`ui/`): `desktop` = Electron Forge + React + Radix + Vite (`ui/desktop/package.json:10-69`); `text` = a Node TUI shipped as npm `@aaif/goose`, launched by `goose tui` (`crates/goose-cli/src/cli.rs:1045-1064`); `sdk` = TypeScript ACP client (`ui/sdk/src/goose-client.ts`).
- **Entry points:** `goose-cli/src/main.rs` → `cli.rs` `enum Command` (`cli.rs:801-1075`): Configure, Info, Doctor, Mcp (run bundled MCP servers), Acp, Serve, Session, Run, Recipe, Skills, Plugin, Schedule, Gateway (Telegram bot: `crates/goose/src/gateway/telegram.rs`), Update, Term, Tui, LocalModels.
- **Extension mechanisms:** "extensions" = MCP servers plus in-process variants. `ExtensionConfig` variants: `sse` (deprecated), `stdio`, `builtin` (bundled goose-mcp servers over in-process duplex streams — `crates/goose/src/builtin_extension.rs:5-28`), `platform` (run inside the agent process with direct agent access), `streamable_http` (incl. HTTP-over-Unix-socket), `frontend` (tools executed by the UI), `inline_python` via uvx (`crates/goose/src/agents/extension.rs:160-280`).
- **Built-in "platform" extensions** (`crates/goose/src/agents/platform_extensions/mod.rs:1-200`): developer (shell/edit/tree/image), analyze (tree-sitter code structure), todo, summarize, summon (subagents), orchestrator, chatrecall, apps, ext_manager (search/enable extensions at runtime), tom, optional `code_execution` (pctx code-mode: TypeScript tool chaining, `reply_parts.rs:193-248`).
- **Tests:** inline unit tests + `crates/goose/tests/` integration suites covering ACP server/provider/fork, compaction, MCP integration with replay fixtures, subprocess cleanup, tool-inspection precedence (`crates/goose/tests/` listing). CLI has `scenario_tests`. `evals/` holds `harbor` and `release_risk_check`.
- **Docs:** Docusaurus site, ~170 md files (`documentation/docs`). Release: weekly minor releases via tag-triggered GitHub Actions (`RELEASE.md:1-13`). CI: ~25+ workflows incl. `goose-issue-solver.yml` and `goose-pr-reviewer.yml` — goose runs itself on its own repo; `goose-self-test.yaml` recipe at repo root. Cadence: 273 commits in the month to 2026-08-04 (`git log --oneline --since=2026-07-04 | wc -l`).

Architectural map: UIs (CLI REPL, Electron, npm TUI, Telegram gateway, editors) → ACP (stdio/WebSocket) → `Agent` (crates/goose/src/agents/agent.rs) → ExtensionManager (rmcp MCP clients + platform tools) and Provider trait (goose-providers HTTP formats, ACP/CLI agent providers, local candle inference) → SQLite session store.

## B. One agent turn (CLI path)

1. **Input ingestion:** CLI REPL builds a user `Message`, spawns a ctrl-c watcher that fires a `CancellationToken`, calls `Agent::reply` (`crates/goose-cli/src/session/mod.rs:1302-1328`).
2. **Entry:** `Agent::reply` → `reply_impl` (`crates/goose/src/agents/agent.rs:1637-1657`). Elicitation responses short-circuit (`agent.rs:1671-1706`); `SessionStart`/`UserPromptSubmit` hooks emitted (`agent.rs:1735-1752`); slash commands resolved by `execute_command` (`agent.rs:1754-1756`; `/compact /clear /skills /doctor /status /goal /grind` + recipe/skill slash commands — `agents/execute_commands.rs:11-135`).
3. **Persist input + pre-turn compaction check:** user message stored via `SessionManager::add_message`; `check_if_compaction_needed` runs *before* the loop; if over threshold, `compact_messages` summarizes and `replace_conversation` rewrites history, emitting `AgentEvent::HistoryReplaced` (`agent.rs:1860-1953`).
4. **Context assembly:** `reply_internal` → `prepare_reply_context` (`agent.rs:1962-1986, 789-843`): `fix_conversation` repairs malformed history; `prepare_tools_and_prompt` (`agents/reply_parts.rs:186-295`) lists tools from ExtensionManager, sorts them by name "for multi session prompt caching" (`reply_parts.rs:254-255`), renders `system.md` template (`agents/prompt_manager.rs:162-164`) with extension info + hints. Hints = `.goosehints` + `AGENTS.md` by default, configurable via `CONTEXT_FILE_NAMES` (CLAUDE.md is test-verified) (`hints/load_hints.rs:13-24`, tests at `load_hints.rs:440-474`); global hints from config dir and `~/.agents/AGENTS.md` (`load_hints.rs:237-252`); per-subdirectory hints are hot-loaded mid-session as the agent touches new dirs (`agent.rs:2857-2867`). Project instructions appended per session (`agent.rs:775, 1984-1986`).
5. **Request construction/turn context:** each iteration injects a `<turn-context>` block ("moim") with time, cwd, compaction status, turn budget (`agents/moim.rs:1-38`; `agent.rs:2185-2194`).
6. **Provider call:** `Agent::stream_response_from_provider` (`reply_parts.rs:315-380`) filters to agent-visible messages, applies toolshim (tools-as-text for non-tool-calling models, `reply_parts.rs:283-292,331-336`), calls `provider.stream(...)` — trait `Provider` in `crates/goose-provider-types/src/base.rs:424` (`stream` :429, `complete` :437).
7. **Streaming loop:** `agent.rs:2092-2848`. Usage chunks → `AgentEvent::Usage` (`agent.rs:2249-2257`); content chunks → `categorize_tools` splits frontend vs remaining tool requests (`agent.rs:2291-2301`).
8. **Authorization:** Chat mode skips tools with a canned response (`agent.rs:2360-2376`). Otherwise `ToolInspectionManager.inspect_tools` runs inspectors (permission, security, repetition) then `process_inspection_results_with_permission_inspector` produces approved/needs_approval/denied (`agent.rs:2378-2401`; `permission/permission_inspector.rs:14-90`). Modes: Auto/Approve/SmartApprove/Chat (`goose-provider-types/src/goose_mode.rs:22-31`). SmartApprove read-only detection is **an LLM call** with `permission_judge.md` (`permission/permission_judge.rs:145-185`). Needs-approval tools yield an `ActionRequired` message and block on a oneshot from `tool_confirmation_router`; AlwaysAllow/NeverAllow persists to PermissionManager (`agents/tool_execution.rs:112-204`; store = `~/.config/.../tool_permissions.json`, `permission/permission_store.rs:45-70`).
9. **Execution locus:** `Agent::dispatch_tool_call` (`agent.rs:1127-1260`) — PreToolUse hooks can deny by policy (`agent.rs:1178-1186`), schedule/final-output/frontend tools handled in-agent, everything else → `ExtensionManager::dispatch_tool_call` (`agents/extension_manager.rs:1812-1842`) which resolves the owning extension client (rmcp MCP client or platform client). MCP calls inject session context and honor the CancellationToken (`agents/mcp_client.rs:671-690`).
10. **Observation return:** tool streams multiplexed via `stream::select_all`; results appended as tool-response messages; MCP notifications forwarded as `AgentEvent::McpNotification` (`agent.rs:2435-2503`). Oversized text results (>200k chars, `GOOSE_MAX_TOOL_RESPONSE_SIZE`) are written to a temp file and replaced by a pointer, not truncated (`agents/large_response_handler.rs:5-26,68`).
11. **Model-specific reassembly:** thinking blocks are re-attached to tool-call messages because "Gemini and Kimi/DeepSeek require it echoed... duplicate signed blocks make Anthropic reject the turn with a 400" (`agent.rs:2513-2545`).
12. **Termination:** loop ends when no tools called and no goal/grind/retry/final-output continuation applies (`agent.rs:2889-3013`); max_turns default 1000 (`agent.rs:69,2179-2183`); user "steer" messages drained mid-run (`agent.rs:2097-2122`, `steer` at :544).
13. **Failure paths:** `ContextLengthExceeded` → in-loop compaction (max 2 attempts) (`agent.rs:2723-2781`); CreditsExhausted/Refusal/NetworkError each have dedicated user-facing handling (`agent.rs:2783-2847`); empty responses retried up to `MAX_EMPTY_TURN_RETRIES` then surfaced (`agent.rs:2966-2988`); recipe `retry_config` runs shell success-checks with timeouts (`agents/retry.rs:1-45`).
14. **Persistence:** SQLite via sqlx (WAL) — `sessions`, `messages`, `usage_ledger` tables (`session/session_manager.rs:19-20,888,960-1016`); every message persisted as produced; compaction is *metadata-based visibility flipping* plus summary insertion (`agent.rs:3022-3049` marks tool pairs agent-invisible; `context_mgmt/mod.rs:26-80`). Stop hooks can veto session end and force continuation, capped (`agent.rs:2137-2169`).

## C. Matrix inputs

| Dimension | Label | Note / citation |
|---|---|---|
| TUI/CLI | native | REPL `goose session` + separate Node TUI `goose tui` (`cli.rs:892-944,1045-1064`) |
| IDE integration | via-ACP/external | goose is an ACP agent server (`acp/server.rs:1427-1485`); no in-repo VS Code plugin found |
| Desktop/web UI | native | Electron app is an ACP WebSocket client (`ui/desktop/src/main.ts:1123`) |
| Headless/CI | native | `goose run -i FILE|-t TEXT --no-session -q --output-format stream-json` (`cli.rs:212-234,295-313,346-392`) |
| Server API | native | `goose serve` = ACP over HTTP/WS + TLS + secret key (`cli.rs:843-890`; auth `acp/transport/auth.rs:15-36`) |
| SDK | native | `goose-sdk` (Rust + uniffi Python/Kotlin), `ui/sdk` TS (`goose-sdk/src/lib.rs:1-13`) |
| Multi-session | native | SQLite SessionManager, session list/close via ACP (`acp/server.rs:1463-1469`) |
| Session branching | native | `goose session --resume --fork`, plus `--edit` history in $EDITOR (`cli.rs:913-929`); ACP fork test (`tests/acp_fork_session_test.rs`) |
| Approval UX | native | ActionRequired message + confirmation channel (`tool_execution.rs:134-165`) |
| Diff/checkpoint UX | partial | edit tool returns previews/errors (`developer/edit.rs:157-196`); `agents/snapshots` dir exists — UNVERIFIED: no checkpoint/restore verified read |
| Provider abstraction | native | trait `Provider` (`goose-provider-types/src/base.rs:424-437`); leaks: see E |
| OpenAI-compat + local | native | `OPENAI_HOST`/`OPENAI_BASE_PATH` (`goose-providers/src/openai.rs:645-652`); 35+ declarative providers compiled from definitions incl. lmstudio, ollama; local candle inference crate (`goose-providers/src/declarative.rs:10-55`) |
| Model-specific adaptations | native | thinking re-attachment per model family (`agent.rs:2513-2521`); toolshim for non-tool models (`reply_parts.rs:283-292`) |
| Reasoning-model support | native | Thinking/RedactedThinking content types, `with_default_thinking_effort` (`reply_parts.rs:353-354`) |
| Prompt caching | native | anthropic `cache_control: ephemeral` on tools tail + message tail (`goose-provider-types/src/formats/anthropic.rs:409-432,514-531`); stable tool ordering for cache (`reply_parts.rs:254-255`) |
| Token/cost accounting | native | Usage events per chunk (`agent.rs:2249-2257`); `usage_ledger` table + `accumulated_cost` (`session_manager.rs:1016,77`) |
| Retry/fallback | native | provider retry trait (`goose-provider-types/src/retry.rs:155`); empty-turn retries; recipe retry w/ success checks (`retry.rs:1-45`) |
| Drive another agent | **native** | ACP *client* provider spawns external agent processes — claude_acp, codex_acp, copilot_acp, amp_acp, pi_acp; CLI wrappers claude_code, codex, gemini_cli, cursor_agent, kimicode (`providers/` listing; `acp/provider.rs:50-68,1029-1046`); `opencode_go` declarative provider (`declarative.rs:40`) |
| Built-in tools | native | developer shell/edit/tree, analyze, todo, subagents, extension search (`platform_extensions/mod.rs:28-200`) |
| Tool registration/schemas | native | rmcp `Tool` JSONSchema; schema normalization (`agents/tool_schema_normalize.rs`) |
| MCP client | native | rmcp 3.0.0 (`Cargo.toml:22`); handshake `McpClient::connect_with_container` → `client.serve(transport)` (`agents/mcp_client.rs:612-648`). Spec revision: negotiated by rmcp; UNVERIFIED exact date string (rmcp source not vendored; lockfile also carries transitive rmcp 1.8.0) |
| MCP server | native | bundled servers (memory, computercontroller, autovisualiser, tutorial) via `goose mcp <name>`, rmcp stdio serve (`goose-mcp/src/mcp_server_runner.rs:1-30`) |
| ACP | native, both roles | agent: `acp/server.rs:1427-1485` (`agent-client-protocol` 1.0, schema 1.1, `Cargo.toml:23-25`; `ProtocolVersion::LATEST` in tests `acp/server.rs:2708`); client: `acp/provider.rs` (spawns subprocess, initialize at :1068) |
| LSP | absent | no LSP client found in `crates/goose/src` (grep `lsp` — no hits in core src) |
| Browser/web tools | via-extension | `web_scrape` in computercontroller MCP (`goose-mcp/src/computercontroller/mod.rs:602-611`); no CDP/playwright driver in core |
| Shell execution | native | developer platform ext; login-shell resolution, flatpak host escape, streamed output batching (`developer/shell.rs:31-52,363-379`) |
| File edit | native | `string_replace` exact-string, single-match required; on failure returns "Did you mean" similar-context + file preview — model-mediated recovery, no fuzzy auto-apply (`developer/edit.rs:157-196`) |
| Git integration | partial | no git tool; shell-based; git-command security patterns tested (`tests/git_command_security.rs`, exists) |
| Custom-tool API | native | any ExtensionConfig type; runtime enable via `manage_extensions` tool persisted to session (`agent.rs:2505-2510`) |
| Extension lifecycle | native | ExtensionManager add/remove, OSV malware check of stdio commands against osv.dev (`agents/extension_malware_check.rs:14-23`) |
| Tool-result truncation | native | >200k chars → spill to file, not truncated (`large_response_handler.rs:5-26`) |
| Repo maps | partial | analyze ext: tree-sitter overviews/call graphs on demand (`platform_extensions/mod.rs:32-43`); no persistent map |
| Search/retrieval | partial | chatrecall (chat history search `session/chat_history_search.rs`); code search via shell |
| Agent Skills | native | SKILL.md + frontmatter per agentskills.io spec; `~/.agents/skills` and `<project>/.agents/skills` (`skills/mod.rs:24-47`); skills as slash commands (`slash_commands/skill_slash_command.rs:8-43`) |
| AGENTS.md/.goosehints | native | both, configurable, nested, global (`hints/load_hints.rs:10-24,237-252`) |
| Memory | via-extension | goose-mcp `memory` server (`mcp_server_runner.rs:10-21`) |
| Compaction | native | threshold 0.8 default; LLM summary + visibility metadata (original rows retained in DB); separate incremental tool-pair summarization (`context_mgmt/mod.rs:26-49`; `agent.rs:3022-3049`) |
| Plan/build modes | partial | `plan.md` prompt exists (`prompts/` listing); Chat mode = no-tools planning (`goose_mode.rs:30-31`); no distinct plan/apply state machine found |
| Todos | native | todo platform extension (`platform_extensions/mod.rs:46-53`) |
| Subagents | native, real | `run_subagent_task` constructs a real `Agent::with_config` and drives `reply` with own session + recipe (`agents/subagent_handler.rs:121-188`) |
| Recipes | native | YAML: version/title/description/instructions/prompt/extensions/settings/parameters/response(json_schema)/sub_recipes/retry (`recipe/mod.rs:42-207`) |
| Parallelism | native | tool futures run via `select_all` concurrently (`agent.rs:2435-2503`); subagent execution tool |
| Background execution | partial | scheduler sessions; steering while running; no detached shell-job manager verified |
| Scheduling | native | tokio_cron_scheduler in-process, `goose schedule` CRUD + `RunNow` (`scheduler.rs:12,216-219`; `cli.rs:607-670`) |
| Sandbox | partial | host exec by default; opt-in Docker container for extensions (`agents/container.rs:1-15`; `cli.rs:135`); flatpak detection escapes *out of* sandbox (`developer/shell.rs:31-38`) |
| Credentials | native | system keyring w/ fallback file; `GOOSE_DISABLE_KEYRING` (`config/base.rs:5-6,92-100`) |
| Permission model | native but prompt-level | allow/deny/ask per tool name; readonly annotations; **no OS enforcement** — see E |
| Prompt-injection defenses | native (heuristic+LLM) | PromptInjectionScanner, adversary/egress inspectors, user-decision telemetry (`security/mod.rs:24-50`; `tool_execution.rs:149-163`) |
| Verification loop | native | `goose review` + `.agents/checks/*.md` + `REVIEW.md` run as check subagents (`checks/mod.rs:1-45`); recipe success-check shell commands (`retry.rs`) |
| Hooks/events | native | 11 HookEvents incl. PreToolUse deny + Stop veto (`hooks/mod.rs:50-62`; `agent.rs:1156-1186,2137-2169`) |
| Config portability | native | YAML config + env vars; secrets in keyring; declarative provider defs |
| Setup cost | low-med | `goose configure` wizard; desktop installer; provider signup flows in-repo (`config/signup_openrouter`) |
| Effort: add tool | low | any MCP server, or SKILL.md file |
| Effort: add provider | low | declarative YAML definition compiled in or custom OpenAI-compat config (`declarative.rs:10-55`) |
| Effort: change loop | high | 4,917-line `agent.rs`; loop is one giant `async_stream::try_stream!` |
| Effort: embed | low-med | goose-sdk uniffi / ACP client; `Agent` is a library type in crate `goose` |

## D. Governing philosophy

Open/flexible/choice are written into governance (`GOVERNANCE.md:8-13`) and visible in code: the provider layer treats *other coding agents as models* (ACP/CLI providers), the client layer treats *editors and its own desktop app as ACP clients*, and everything tool-shaped is MCP. goose **owns** the hard middle: conversation repair (`fix_conversation`), compaction (three mechanisms: pre-turn, in-loop recovery, incremental tool-pair summarization), model-family quirks (thinking re-attachment, toolshim), retries, scheduling, telemetry, security inspection. It **pushes onto extensions/users**: sandboxing (host exec default; Docker opt-in), long-term memory (an MCP server), browser automation, and evaluation of what is "safe" (LLM permission judge, LLM injection scanner). Standards-maximalism is the bet: ACP as its own API replaced a bespoke REST server, at the cost of coupling product features to protocol `_meta` extensions (goose custom capabilities in `acp/server.rs:1437-1455`).

## E. Hidden tradeoffs

- **CONFIRMED — permission boundaries are process-internal, not OS-enforced.** Approval is a message + oneshot channel in the same process (`tool_execution.rs:134-147`); shell runs on host with login shell (`developer/shell.rs:332-379`); an approved `shell` call can do anything, and SmartApprove's read-only classification is itself an LLM completion (`permission_judge.rs:145-185`) — a misclassification auto-approves a write.
- **CONFIRMED — flatpak sandbox is deliberately escaped** for shell (`flatpak-spawn --host`, `developer/shell.rs:31-52`): packaging sandbox ≠ agent sandbox.
- **CONFIRMED — model-family dependence inside the core loop:** Anthropic-400/Gemini/Kimi thinking-block surgery lives in agent.rs, not the provider layer (`agent.rs:2513-2545`) — leaky abstraction by the project's own comments.
- **CONFIRMED — compaction is lossy by design at the model-visible layer** (summary replaces content; continuation prompts instruct the model to hide that compaction happened, `context_mgmt/mod.rs:36-49`), though original messages persist in SQLite as agent-invisible rows (`agent.rs:3038-3044`).
- **CONFIRMED — auto-compaction can fire twice per user turn** (pre-turn at `agent.rs:1874-1953`, in-loop at :2723-2768) plus background tool-pair summarization — three token-spending LLM passes a benchmark must account for.
- **CONFIRMED — the core loop is a 1,000+ line single try_stream** (`agent.rs:2067-3050`) with ~15 loop-carried mutable flags (`agent.rs:2068-2090`); correctness depends on flag interplay (empty_response, retrying_after_*, goal_check_pending…). Tests exist around it (`tests/agent.rs`, `compaction.rs`) but the state machine is implicit.
- **CONFIRMED — 100ms polling in tool-result multiplexing** (`tokio::select!` with `sleep(100ms)` arm, `agent.rs:2450-2501`).
- **CONFIRMED — hidden steering prompts:** goal/grind/final-output nudges are injected as *user-role* messages marked user-invisible (`agent.rs:2900-2947`); transcript in DB ≠ what user saw ≠ what model saw (three visibility projections).
- **CONFIRMED — telemetry on by default paths:** posthog + OTel gen_ai spans capture message content when enabled (`agent.rs:1664-1669`; `posthog.rs`); relevant for privacy-sensitive benchmarking.
- **RISK — prompt-injection defense is best-effort:** scanner+LLM classification with env-var overrides (`security/mod.rs:17-22,45-50`); a determined injection that convinces the judge passes; no egress *blocking* found, only inspection.
- **RISK — recipe/skill portability:** recipes are goose-schema YAML (`recipe/mod.rs:42-85`) with goose-specific settings/extension blocks; skills follow agentskills.io and are portable, but recipe `sub_recipes`/`retry`/`response` are harness-locked.
- **RISK — cancellation of MCP tools depends on server cooperation:** CancellationToken is honored on goose's side of the request (`mcp_client.rs:671-690`), but a stdio server mid-side-effect isn't rolled back.

## F. Interop surfaces

- **MCP client handshake:** `crates/goose/src/agents/mcp_client.rs:612-648` (`GooseClient` → rmcp `client.serve(transport)`; peer protocol version captured at :639-641). rmcp **3.0.0** direct dependency (`Cargo.toml:22`; lockfile also shows transitive rmcp 1.8.0). Client info/capabilities at `mcp_client.rs:326-343,553-564` (advertises MCP Apps UI extension per test :1476). UNVERIFIED: exact spec-date string of rmcp 3.0.0's LATEST.
- **MCP server:** bundled servers served with rmcp stdio in `crates/goose-mcp/src/mcp_server_runner.rs:1-30` (`goose mcp memory|computercontroller|autovisualiser|tutorial`).
- **ACP agent (server) role:** `crates/goose/src/acp/server.rs:1427-1485` `on_initialize` — crate `agent-client-protocol` 1.0 / schema 1.1 unstable (`Cargo.toml:23-25`); transports: stdio (`goose acp`) and HTTP/WebSocket with `X-Secret-Key` auth (`acp/transport/auth.rs:15-36`); session load/list/close, fork tested (`tests/acp_fork_session_test.rs`).
- **ACP client role (goose drives other agents):** `crates/goose/src/acp/provider.rs` — `AcpProviderConfig{command,args,env,mcp_servers,mode_mapping…}` (:50-68), `spawn_acp_process` (:1029-1046), initialize with `ProtocolVersion::LATEST` (:1068). Concrete provider defs: `providers/claude_acp.rs`, `codex_acp.rs`, `copilot_acp.rs`, `amp_acp.rs`, `pi_acp.rs`; CLI-subprocess providers `claude_code.rs`, `codex.rs`, `gemini_cli.rs`, `cursor_agent.rs`, `kimicode.rs` (dir listing; README.md:31 markets "use your existing Claude, ChatGPT, or Gemini subscriptions via ACP").
- **Session import from other harnesses:** `goose session import` accepts "Claude Code / Codex / Pi .jsonl" (`cli.rs:585`; `session/import_formats/`).
- **Desktop/TUI/SDK are ACP clients** (`ui/desktop/src/main.ts:1123`; `goose-sdk/src/lib.rs:1-13`).

## G. Benchmark inputs

- Headless: `goose run -t "..."` or `-i file|-` (stdin), `--no-session`, `-q`, `--output-format stream-json`, `--max-turns` via SessionOptions, model override via ModelOptions (`cli.rs:212-234,295-313,346-392,948-969`).
- Full-auto: `GOOSE_MODE=auto` config/env (`config/base.rs:1109`); per-run recipe `settings.goose_provider/goose_model` (`recipe/mod.rs:98-110`).
- Fixed endpoint: `OPENAI_HOST` + `OPENAI_BASE_PATH` (`openai.rs:645-652`) or a declarative provider definition; local: ollama/lmstudio declaratives, built-in candle inference.
- Approval scripting: auto mode bypasses prompts; hooks (PreToolUse deny) allow policy scripting (`agent.rs:1178-1186`).
- Fairness blockers: three auto-LLM side-calls (compaction x2 paths, tool-pair summarization, session naming `agent.rs:2005-2024`, permission judge in smart mode) consume tokens invisibly — disable via `GOOSE_TOOL_PAIR_SUMMARIZATION=false`, high `GOOSE_AUTO_COMPACT_THRESHOLD`, `disable_session_naming` config, mode=auto; moim turn-context injection adds per-turn tokens (skippable only via internal `SKIP` thread-local, `moim.rs:27-29` — not user-exposed: RISK); default max_turns 1000 is benchmark-friendly.

## H. Portability-adapter inputs

- **Skills=procedure:** direct fit — goose implements agentskills.io SKILL.md discovery in `~/.agents/skills` and `<project>/.agents/skills` (`skills/mod.rs:24-47`); zero adapter needed.
- **AGENTS.md=policy:** direct fit — AGENTS.md is a default hint filename incl. nested and global `~/.agents/AGENTS.md` (`load_hints.rs:13-24,248`).
- **Files=state / git=history:** developer shell+edit suffice; no harness state to fight except the SQLite session store (ignorable).
- **MCP=optional tools:** first-class, all transports (`extension.rs:160-280`).
- **Thin adapter surface:** a recipe YAML (`recipe/mod.rs:42-85`) wrapping the methodology prompt, with `sub_recipes` per phase and `retry.checks` shell success-gates (`agents/retry.rs`) is the native way to run spec→plan→tests→implement→review→verify unattended; `goose review` checks (`checks/mod.rs:1-45`, `.agents/checks/*.md`) map directly to a review phase and are themselves file-based/portable-ish.
- **What goose silently fights:** auto-compaction rewriting history mid-methodology (visibility flips could hide the spec from the model — mitigate with high threshold); built-in todo/plan prompts overlapping a file-based todo convention; goal/grind hidden nudges if enabled; session naming and moim token overhead; SmartApprove LLM judge stalling unattended runs (use auto mode); Stop hooks could be repurposed as verification gates (`agent.rs:2137-2169`) — a positive.
- **Unattended methodology coverage:** high — headless run + recipes + sub_recipes + retry success-checks + review checks + scheduler cover spec→…→verify; PR creation needs `gh` via shell (no git/PR tool).

**Strengths:** deepest protocol composition of the cohort (MCP both roles, ACP both roles, other agents as providers); real subagents; robust failure taxonomy in the loop; sessions in SQLite with fork/edit/import; recipes+scheduler+review = automation platform; weekly releases, self-dogfooding CI.
**Weaknesses:** monolithic loop; prompt-level security (LLM-judged permissions); host execution default; hidden LLM side-calls; model-family patches in core; desktop/TUI/SDK all hinge on unstable ACP `_meta` extensions.
**Best fit:** multi-provider automation, agent-of-agents orchestration, local/OSS-model workflows, scheduled/recipe-driven ops.
**Poor fit:** hard-sandbox enterprise policy enforcement; minimal-token benchmark baselines without configuration; embedding where a small auditable loop is required.
