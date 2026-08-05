# 06 — badlogic/pi-mono (`pi` coding agent + harness construction kit)

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commit below.*

Follow-up work (traces not performed in this study): fetch author's "what if you don't need MCP" and pi blog posts for external-position quotes; read harness-v2.md Parts II–III fully; run `pi --mode json` smoke test; inspect pi-tui differential renderer.

Subject pinned at commit `588915ec71714688cee8b7153339e8bdebb3e82e` (verified via `git rev-parse HEAD`).
Provisional classification **verified with one amendment**: it *is* a minimal-core, extensible TypeScript harness construction kit (pi CLI + agent runtime + provider layer + TUI as separate npm packages), but "minimal" applies to the *feature surface*, not the codebase — the repo is ~5,400 commits, 409 test files, ~45 provider integrations, and is growing an experimental durable-server layer (protocol/client/server/sqlite packages).

## A. Orientation

**Monorepo layout** (npm workspaces, all packages lockstep-versioned at 0.83.0; root package.json workspaces list):

| Package | Role | Deps (internal) |
|---|---|---|
| `@earendil-works/pi-ai` (`packages/ai`) | Unified multi-provider LLM streaming API, model catalog, OAuth flows | — |
| `@earendil-works/pi-agent-core` (`packages/agent`) | Agent loop, `Agent` class, tool execution, event streaming; new `harness/` durable-harness code | pi-ai |
| `@earendil-works/pi-tui` (`packages/tui`) | Terminal UI lib, differential rendering; only external deps `get-east-asian-width`, `marked` | — |
| `@earendil-works/pi-coding-agent` (`packages/coding-agent`) | The `pi` CLI: tools, sessions, extensions, skills, modes (interactive/print/json/rpc), SDK | agent-core, ai, tui, client, protocol |
| `@earendil-works/pi-protocol` / `pi-client` / `pi-server` | Experimental transport-neutral CBOR protocol for remote pi sessions ("may change or be removed without notice", packages/server/README.md:3) | protocol |
| `@earendil-works/pi-storage-sqlite-node` (`packages/storage/sqlite-node`) | SQLite session backend for harness v2 | agent-core |
| `@earendil-works/pi-evals` | Model-backed behavioral evals over real `AgentSession` via vitest-evals | coding-agent |

Dependency direction is clean: ai ← agent ← coding-agent; tui is only consumed by coding-agent (and its tool renderers). packages/agent/package.json deps: `@earendil-works/pi-ai, diff, ignore, typebox, yaml` — the runtime core has no TUI dependency.

- **Language/runtime**: TypeScript (Node ≥22.19, erasable-syntax-only style), Bun used to compile standalone binaries (README.md:62-73). Biome lint, tsgo typecheck.
- **Entry points**: `packages/coding-agent/src/main.ts` (CLI), `src/core/sdk.ts` (`createAgentSession`), `src/rpc-entry.ts`; library entry `packages/agent/src/index.ts`.
- **User surfaces**: interactive TUI; `pi -p` print; `--mode json` event stream; `--mode rpc` JSONL stdin/stdout; SDK; `pi install/update/list/config` package manager.
- **Extension mechanisms**: TypeScript extensions (`~/.pi/agent/extensions/`, `.pi/extensions/`), Agent Skills, prompt templates, themes, "pi packages" distributed via npm/git (`pi install npm:…`).
- **Tests**: 409 `*.test.ts` files (212 coding-agent, 125 ai, 30 tui, 15 agent). Test policy: faux provider for suite tests, no paid tokens (AGENTS.md:32); e2e tests activate only with API keys. CI (`.github/workflows/ci.yml`): `npm ci --ignore-scripts && build && check && test`, plus npm-audit, publish, issue-gate workflows.
- **Docs**: 31 markdown docs under `packages/coding-agent/docs/` (12,175 lines), incl. rpc.md (1,578), extensions.md (2,987), sdk.md (1,205). Design RFCs external at rfc.earendil.com.
- **Release/versioning**: lockstep versions, "patch = fixes + additions, minor = breaking. No major releases" (AGENTS.md:123); npm trusted publishing via GitHub OIDC; standalone Bun binaries; documented supply-chain hardening (pinned exact deps, shrinkwrap, `min-release-age=2`) (README.md:75-87).
- **License**: MIT, copyright Mario Zechner (LICENSE:1-2).
- **Maturity/bus factor**: first commit 2025-08-09; ~5,420 commits in 12 months, 400–500/month sustained recently (git log). Mario Zechner 3,502 commits (~65%), Armin Ronacher 472, ~8 further contributors with 40+ — dominant-author but not strictly single-author. New-contributor issues/PRs are **auto-closed by default** and reviewed daily (README.md:11) — deliberate contribution throttle.
- **Product coupling**: none to a paid service; org moved to `earendil-works`; sibling `pi-chat` repo for Slack; author publishes his own sessions as open datasets (README.md:89-103).

## B. One agent turn, keystroke → persisted state (interactive mode)

1. **Input ingestion** — pi-tui editor inside `InteractiveMode`; main loop `const userInput = await this.getUserInput(); await this.session.prompt(userInput)` (packages/coding-agent/src/modes/interactive/interactive-mode.ts:1068-1076; getUserInput resolves an `onInputCallback` at 3694-3706).
2. **Pre-processing in `AgentSession.prompt()`** (packages/coding-agent/src/core/agent-session.ts:1116-1272): extension slash-command dispatch (1124-1131) → `input` extension event may handle/transform the text (1142-1157) → `/skill:name` expansion inlines SKILL.md into `<skill …>` block (1309-1333) → prompt-template expansion → if streaming, queue as steer/followUp (1166-1180) → model + auth preflight (1186-1203) → compaction check (1205-1210).
3. **System/project-instruction assembly** — `buildSystemPrompt()` (packages/coding-agent/src/core/system-prompt.ts:28-162): a short default prompt ("You are an expert coding assistant operating inside pi…", 121-138) + one-line tool list + guidelines + **paths to pi's own docs so the model can explain/extend pi** (131-138) + `<project_context>` wrapping AGENTS.md/CLAUDE.md content (145-152; candidates `["AGENTS.md","AGENTS.MD","CLAUDE.md","CLAUDE.MD"]`, resource-loader.ts:71, with ancestor inheritance and git-worktree shadow handling, resource-loader.ts:90-110) + skills XML (154-157) + cwd.
4. **Extension pre-turn hook** — `before_agent_start` event may inject custom messages and override the system prompt per turn (agent-session.ts:1233-1261).
5. **Loop entry** — `_runAgentPrompt` → `Agent.prompt()` (packages/agent/src/agent.ts:344-356) → `runAgentLoop()` (packages/agent/src/agent-loop.ts:95-118). Outer loop drains follow-up queue; inner loop drains steering messages between turns (agent-loop.ts:167-272).
6. **Context selection / request construction** — `streamAssistantResponse()` (agent-loop.ts:281-312): optional `config.transformContext` (AgentMessage[]→AgentMessage[]) then required `config.convertToLlm` (AgentMessage[]→provider Message[]; default in coding-agent at core/messages.ts:148) then `Context {systemPrompt, messages, tools}`.
7. **Provider abstraction** — a single injected `StreamFn` (agent/src/stream-fn.ts:11-20); coding-agent installs `ModelRuntime`'s pi-ai `streamSimple`. pi-ai routes by `model.api` to per-API modules (packages/ai/src/api/: anthropic-messages, openai-completions, openai-responses, openai-codex-responses, azure, google-generative-ai, google-vertex, bedrock-converse-stream, mistral-conversations, pi-messages), across ~45 provider configs (packages/ai/src/providers/).
8. **Streaming** — provider events (`text_delta`, `thinking_delta`, `toolcall_delta`, …) update a partial `AssistantMessage` placed in context and re-emitted as `message_update` (agent-loop.ts:317-361).
9. **Tool-call parsing/validation** — `validateToolArguments` (TypeBox schemas) inside `prepareToolCall` (agent-loop.ts:617-618); streamed args finalized with a best-effort JSON salvage parser, so `stopReason === "length"` fails **all** tool calls in the message rather than executing possibly-truncated args (agent-loop.ts:208-214, 381-406).
10. **Authorization** — no built-in gate. `config.beforeToolCall` (wired to the `tool_call` extension event) may `{block: true, reason}` (agent-loop.ts:619-643). This is code-enforced pre-execution, but only exists if the user installs an extension.
11. **Execution locus** — in-process, host permissions. `bash` spawns the user's shell detached, streams output, kills the whole process tree on abort/timeout (core/tools/bash.ts:97-146). Parallel execution by default; per-tool `executionMode: "sequential"` forces sequential batch (agent-loop.ts:418-425, 489-554).
12. **Observation return** — `afterToolCall` hook can rewrite result/isError (agent-loop.ts:720-747); `ToolResultMessage` appended to context (agent-loop.ts:218-221); bash/read results truncated at 2,000 lines / 50 KB with full output spilled to a file path in `details` (core/tools/truncate.ts:11-12).
13. **Loop termination** — no more tool calls and both queues empty → `agent_end`; also `shouldStopAfterTurn` and all-tools-`terminate` short-circuits (agent-loop.ts:247-274, 582-584).
14. **Persistence** — `AgentSession` subscribes to agent events; on every `message_end` it calls `SessionManager.appendMessage` (agent-session.ts:640-656), which `appendFileSync`s one JSONL line with `id`/`parentId` forming a tree in `~/.pi/agent/sessions/--<cwd>--/<ts>_<uuid>.jsonl` (session-manager.ts:1021-1065; docs/session-format.md:1-11). Append-only; branching moves a leaf pointer, never rewrites (session-manager.ts:847-855).
15. **Compaction** — auto when `contextTokens > contextWindow - reserveTokens` (16,384 default; keep last ~20k tokens) → LLM summary appended as `CompactionEntry` with `firstKeptEntryId`, session reloads (compaction/compaction.ts:134-135,237; docs/compaction.md:28-44).
16. **Failure/retry/cancel** — pi-ai reimplements the OpenAI/Anthropic SDK retry policy with interruptible backoff (SDKs called with `maxRetries: 0`) honoring retry-after headers (packages/ai/src/utils/provider-retry.ts:22-105); session-level `auto_retry_start/_end` events with attempt counters (agent-session.ts:166-167); AbortSignal threads through streaming and tool execution; bash timeout is opt-in per call (bash.ts:41-43, no default timeout).

## C. Matrix inputs

| Dimension | Label | Note / citation |
|---|---|---|
| TUI | native | pi-tui differential renderer + InteractiveMode (packages/tui; modes/interactive/) |
| IDE integration | absent | no editor plugin in repo; RPC mode documented as the embedding path for "applications, IDEs, or custom UIs" (docs/rpc.md:3) |
| Desktop/web UI | absent | Slack/chat lives in separate `pi-chat` repo (README.md:35) |
| Headless/CI | native | `pi -p`, piped stdin merged into prompt, `--mode json` event stream (coding-agent README modes table; modes/print-mode.ts:1-8) |
| Server API | partial/experimental | `pi-server` CBOR session server, "no standalone CLI or coding-agent backend" (packages/server/README.md:1-5,37) |
| SDK/embeddability | native (its pitch) | `createAgentSession()` ~6 lines to a working agent (docs/sdk.md:19-35); 13 graded SDK examples (examples/sdk/01-minimal.ts…13-session-runtime.ts); loop itself reusable via `Agent`/`runAgentLoop` with injected `streamFn`, `convertToLlm`, hooks (agent/src/types.ts:173-286) |
| Multi-session | partial | one session per process; concurrent pi processes in same cwd are an explicitly supported workflow with git hygiene rules (AGENTS.md:48-60); harness-v2 "lanes" design adds in-process parallel lanes (packages/agent/docs/harness-v2.md:24) |
| Session branching | native, tree | JSONL entries with `id`/`parentId`, in-place branching, `/tree` navigation + branch summarization, `--fork` (docs/session-format.md:3; session-manager.ts:847-855) |
| Approval UX | absent by philosophy → via-extension | "No permission popups. Run in a container, or build your own confirmation flow with extensions" (coding-agent README:500); examples/extensions/permission-gate.ts, confirm-destructive.ts |
| Provider abstraction & leaks | native, leaks explicit | unified Message/Context/StreamFn; leaks surfaced as declared `compat` flags (supportsDeveloperRole, reasoning formats per vendor — ai/src/types.ts:539-555) rather than hidden |
| OpenAI-compatible + local | native | `~/.pi/agent/models.json` custom providers (Ollama/vLLM/LM Studio) (docs/models.md:17-42); dedicated llama.cpp router integration (docs/llama-cpp.md) |
| Model-specific adaptations | native | per-API modules incl. openai-codex-responses, github-copilot-headers, bedrock, vertex; per-model compat overrides (docs/models.md:38-46) |
| Reasoning models | native | thinking levels off…max/xhigh, per-level token budgets, provider-specific reasoning encodings (ai/src/types.ts:555; docs/settings.md thinkingBudgets) |
| Prompt caching | native | Anthropic `cache_control` on last user message + last tool (ai/src/api/anthropic-messages.ts:1256-1273,1320); OpenAI `prompt_cache_key` from session id (openai-codex-responses.ts:562); cache-miss notices setting (docs/settings.md:34) |
| Token/cost accounting | native | every AssistantMessage carries Usage incl. per-direction cost (docs/session-format.md:105-120); model cost/contextWindow in catalog (ai/src/types.ts:779-793) |
| Retry/fallback | retry native; fallback absent | provider-retry.ts:22-105; auto_retry events; UNVERIFIED: no automatic cross-model fallback found (only startup model-fallback message, interactive-mode.ts:1040) |
| Drive another complete agent | via-extension/external | subagent example spawns separate `pi` processes with isolated contexts, parallel streaming, cost roll-up (examples/extensions/subagent/README.md:1-12); pi itself drivable via RPC/SDK |
| Built-in tools | native, 7 | `read, bash, edit, write, grep, find, ls` (coding-agent README tool options table) |
| Tool registration/schemas | native | `pi.registerTool()` with TypeBox schema, streamed partial results, custom TUI renderers (docs/extensions.md:73-92) |
| MCP | absent by philosophy | "No MCP. Build CLI tools with READMEs …, or build an extension that adds MCP support" + link to author's essay (coding-agent README:496); "intentionally does not include built-in MCP" (docs/usage.md:301) |
| ACP | absent | zero matches repo-wide (grep) |
| LSP | absent | no LSP client; only prose mentions of "language servers" as ordinary processes (docs/security.md:33) |
| Browser/web tools | absent | no fetch/browser tool in core/tools/ (dir listing); do via bash/curl or extension |
| Shell execution | native | detached spawn of user shell, streamed output, process-tree kill, opt-in timeout (core/tools/bash.ts:97-146); pluggable `BashOperations` for SSH/VM backends (bash.ts:56-75) |
| File-edit mechanism | native | oldText/newText exact match required unique; fuzzy fallback normalizes trailing whitespace, NFKC, smart quotes, Unicode dashes/spaces (core/tools/edit-diff.ts:27-53, 201-224); emits display diff + unified patch (edit.ts:62-69) |
| Git integration | absent (bash only) | no built-in git tools; examples git-checkpoint.ts, auto-commit-on-exit.ts |
| Custom-tool API | native | ToolDefinition = AgentTool + prompt metadata + renderer (core/tools/tool-definition-wrapper.ts:5-28) |
| Extension lifecycle | native | discovery dirs, project-trust gating, hot `/reload`, events (session_start, input, before_agent_start, tool_call, agent_end, project_trust, compaction…), `registerProvider`, `sendMessage`, `appendEntry`, `setActiveTools`, custom UI (extensions/types.ts:1200-1427; docs/extensions.md:5-30) |
| Tool-result truncation | native | 2,000 lines / 50 KB defaults, full output path preserved in details (core/tools/truncate.ts:11-12; bash.ts BashToolDetails) |
| Repo maps | absent | no codebase-map/index anywhere (core dir listing) |
| Search | native | grep tool shells out to ripgrep when available with JS fallback ops (core/tools/grep.ts:64) |
| Agent Skills | native, standard | implements agentskills.io spec, lenient validation; can mount `~/.claude/skills`, `~/.codex/skills` (docs/skills.md:7, 45-56); progressive disclosure via read tool (skills.md:66-72) |
| AGENTS.md | native | AGENTS.md/CLAUDE.md discovery incl. ancestors + worktree shadowing (resource-loader.ts:71,90-110); loaded regardless of project trust (docs/security.md:25) |
| Memory | absent | sessions only; no memory store (no matches) |
| Compaction | native | summary + last ~20k tokens kept; cumulative file-op tracking; iterative re-summarization keeps previously-kept spans (docs/compaction.md:80-84); lost: verbatim old messages (summary replaces them) |
| Plan modes | absent by philosophy | README:502; example extension examples/extensions/plan-mode/ |
| Todos | absent by philosophy | "They confuse models. Use a TODO.md file" (README:504) |
| Subagents | absent by philosophy → via-extension | README:498; examples/extensions/subagent/ (scout/planner/reviewer/worker over pi RPC processes) |
| Parallelism | partial | parallel tool calls default (agent-loop.ts:489-554); no built-in parallel agents (see subagents) |
| Background execution | absent by philosophy | "No background bash. Use tmux." (README:506) |
| Sandboxing | absent — external OS boundary | "Pi does not include a built-in permission system" (root README.md:39); "No Built-in Sandbox … Real isolation needs to come from the operating system or a virtualization/container boundary" (docs/security.md:31-35); patterns: Gondolin micro-VM tool routing, Docker, OpenShell (docs/containerization.md:12-18) |
| Credential handling | native | `~/.pi/agent/auth.json` written mode 0o600 with chmod enforcement (core/auth-storage.ts:23,67); OAuth auto-refresh for Codex/Claude/Copilot/xAI/OpenRouter (docs/providers.md:17-27) |
| Permission model | none OS-enforced | project trust gates only *resource loading* (settings/extensions), not tool actions (docs/security.md:7,37); extension `tool_call` blocks are runtime-enforced in the loop (agent-loop.ts:636-642) but optional; nothing is OS-enforced in-process |
| Prompt-injection defenses | absent, documented | "Prompt injection … is expected local-agent risk and cannot be reliably prevented by pi" (docs/security.md:37) |
| Verification loop | partial | exit codes/tool errors fed back; evals package exists for harness developers (packages/evals/README.md:1-5); no built-in verify step |
| Hooks/events | native | full AgentSessionEvent stream mirrored to JSON/RPC modes (docs/json.md:1-14; docs/rpc.md events §) |
| Config portability | native | global `~/.pi/agent/settings.json` + project `.pi/settings.json`; keybindings, themes, packages all file-based (docs/settings.md:1-9) |
| Setup cost | low | `npm install -g --ignore-scripts` or curl installer; `/login` for subscription auth (docs/index.md:7-35) |
| Effort: add tool | trivial | ~20-line extension (docs/extensions.md:73-92) |
| Effort: add provider | low | models.json entry for OpenAI-compatible; `pi.registerProvider()` for custom APIs/OAuth (extensions/types.ts:1374-1411; docs/custom-provider.md) |
| Effort: change loop | low-medium | SDK exposes `transformContext`, `convertToLlm`, `beforeToolCall`/`afterToolCall`, `shouldStopAfterTurn`, `prepareNextTurn`, steering/follow-up providers (agent/src/types.ts:173-286) |
| Effort: embed | trivial-to-deep | 6-line embed → full control (`createAgentSessionRuntime`, custom ResourceLoader, in-memory sessions) (docs/sdk.md:19-67) |

## D. Governing philosophy

Explicit, documented minimal-mechanism stance: "Pi is a minimal terminal coding harness. Adapt pi to your workflows, not the other way around, without having to fork and modify pi internals" (coding-agent README:15) and "Pi is aggressively extensible so it doesn't have to dictate your workflow… This keeps the core minimal" (README:494). The absence list is authored, not accidental — each "No X" has a stated rationale and a prescribed alternative (README:496-506; docs/usage.md:299-303). Complexities pi **owns**: provider compatibility matrix (~45 providers, per-vendor reasoning/caching quirks), session durability + tree branching + compaction, streaming TUI, packaging/distribution, supply-chain hardening. Complexities pi **pushes onto users/extensions**: permissions/approvals, sandboxing (OS/container), MCP, subagents, plan/todo workflow, background processes, memory. Distinctive twist: the default system prompt teaches the model where pi's own docs live so the agent can extend itself ("self extensible coding agent", root README.md:15; system-prompt.ts:131-138). The harness-v2 design doc shows deliberate architectural method — a rejected alternative is preserved with reasons (packages/agent/docs/harness-v2.md:3).

## E. Hidden tradeoffs

- **CONFIRMED — No enforced safety rail at all by default**: bash/edit/write run with user permissions immediately; safety depends on user-supplied containers or extensions (root README.md:39; docs/security.md:31-37). Honest, but the default install is the least-guarded of mainstream harnesses.
- **CONFIRMED — Trust prompt guards inputs, not actions**: project trust only gates loading of `.pi` resources; a trusted-or-empty repo can still prompt-inject the model into arbitrary shell commands with zero prompts (docs/security.md:7,37).
- **CONFIRMED — Extensions are arbitrary code with full permissions**, and pi packages install from npm/git; a malicious package = code execution (docs/extensions.md:104; README package security note:409).
- **RISK — Dominant-author bus factor**: 65% of commits by one person; auto-close gate for new contributors limits the pipeline of maintainers (commit-history; README.md:11). Mitigated partially by 470+ commits from a second major contributor.
- **CONFIRMED — Breaking changes are routine**: lockstep "minor = breaking changes. No major releases" policy, and repo rule "Do not preserve backward compatibility unless the user asks" (AGENTS.md:123, 22). Harness-v2 explicitly reserves the right to break everything except v3 JSONL reads (harness-v2.md:5). Teams building on the SDK ride a fast-moving API.
- **CONFIRMED — DIY burden**: approvals, sandboxing, subagents, MCP, plan mode all require building or vetting third-party pi packages (README:496-506). For a team this is real integration and review work the philosophy transfers to them.
- **CONFIRMED — TUI types leak into tool definitions**: built-in tools import pi-tui components for their renderers (core/tools/bash.ts imports `Container, Text` from `@earendil-works/pi-tui`), so `pi-coding-agent` tools carry TUI baggage even in headless embeds; the *agent-core* loop, however, is TUI-free (packages/agent/package.json deps). Renderers are optional at runtime (tool-definition-wrapper.ts:36-47 synthesizes definitions without them).
- **CONFIRMED — No default bash timeout**: a hung command hangs the turn unless the model passed `timeout` or the user aborts (bash.ts:41-43 "no default timeout").
- **RISK — Steering semantics depend on queue draining**: steering messages are only injected between turns (agent-loop.ts:174-190); a long-running tool call delays user correction until it finishes (abort is the only immediate lever).
- **CONFIRMED — Experimental surface inside a "minimal" repo**: protocol/client/server/storage packages and harness-v2 add a second, unstable architecture layer alongside the shipping one (packages/server/README.md:3; harness-v2.md:5) — cognitive load for embedders choosing an API.
- **RISK — model catalog freshness is a build artifact**: provider model JSON is generated at build (`models.generated.ts` header; data files absent from pinned checkout), so vendored builds can drift from live catalogs; mitigated by `pi update --models` and remote catalog refresh (docs/providers.md:3).

## F. Interop surfaces

- **MCP**: no client or server code anywhere in packages/* (grep: only doc mentions). Deliberate: coding-agent README:496 links the author's essay; usage.md:301. An extension can add MCP (stated support path, no first-party example in repo).
- **ACP**: absent (repo-wide grep, zero hits).
- **Embedding pi**: (a) Node SDK — `createAgentSession()`/`AgentSession` (docs/sdk.md:19-35); (b) RPC mode — LF-delimited JSONL over stdin/stdout, full command set incl. prompt/steer/abort/model/compaction/bash/session ops + extension UI protocol (docs/rpc.md:1-40, commands TOC); (c) JSON event mode for one-shot pipelines (docs/json.md:1-8); (d) experimental CBOR remote-session protocol: 4-byte length prefix + CBOR item, version 1, hello handshake (packages/protocol/README.md:5-11) with `PiServer`/`pi-client` (server README:8-37).
- **pi driving others**: nothing built-in; subagent example drives child pi processes via the RPC client (examples/extensions/subagent/README.md:7).
- **Cross-harness resources**: reads Claude Code/Codex skills dirs via settings (docs/skills.md:45-56); loads CLAUDE.md alongside AGENTS.md (resource-loader.ts:71).

## G. Benchmark inputs

- **Headless**: `pi -p "prompt"` (final text), `pi --mode json "prompt"` (full event stream incl. usage/cost per message), stdin piping (README modes table). `--no-session` for ephemeral runs; `--session-dir` to redirect artifacts.
- **Fixed endpoint injection**: `--provider/--model/--api-key` flags; `provider/id:<thinking>` model syntax; custom endpoints via `~/.pi/agent/models.json` (baseUrl + api) — so an OpenAI-compatible benchmark proxy is a config file, no code (docs/models.md:17-36). Env: `PI_PROVIDER`/`PI_MODEL` honored by evals runner (packages/evals/README.md:11-19).
- **Scripting approvals**: nothing to script — no built-in approvals. Project trust in non-interactive modes never prompts; defaults to ignoring project resources unless `--approve`/`-a` (docs/security.md:27; docs/settings.md:15).
- **Determinism aids**: `--no-extensions --no-skills --no-prompt-templates --no-context-files` isolate the bare harness (README resource options); faux provider exists for offline harness tests (ai/src/providers/faux.ts).
- **Fairness blockers**: none serious; note (1) default system prompt embeds pi-doc paths and cwd (minor token overhead), (2) no built-in web/browser tool — web-task benchmarks measure bash+curl, (3) no automatic cross-provider fallback, so rate-limit outages surface as retries/errors, (4) parallel tool calls default-on may differ from serial-only harnesses.

## H. Portability-adapter inputs (Skills=procedure, AGENTS.md=policy, files=state, git=history, MCP=optional tools)

- **Native mechanisms an adapter would use**: AGENTS.md loads natively incl. ancestors (resource-loader.ts:71); Agent Skills per the agentskills.io standard from shared dirs — pi can consume the same `~/.agents/skills`/`.agents/skills` trees as other harnesses (docs/skills.md:25-37); prompt templates as slash commands; extensions API (`pi.registerTool`, `pi.on("tool_call")`, `pi.registerCommand`, `pi.appendEntry` for persisted adapter state — extensions/types.ts:1246-1312) for anything active; `--mode json`/RPC for the outer driver.
- **What pi would silently fight**: essentially nothing found. It has no competing built-in plan/todo/subagent machinery to collide with a methodology's own; skills are only *offered* to the model, though docs note "models don't always do this; use prompting or /skill:name to force it" (docs/skills.md:69) — an adapter should force-load procedure skills via `/skill:name` or template expansion. One caution: no approval layer means the methodology's "review gate" must be an extension or an external orchestrator.
- **Unattended spec→plan→tests→implement→review→verify→PR**: highly feasible headless: each phase = `pi -p`/RPC run with `--no-session` or forked sessions, files as state, git via bash. Steering, compaction, cost accounting and event streams are all exposed over RPC (docs/rpc.md commands: prompt, state, compaction, bash, session). The stock subagent example already encodes scout→planner→worker→reviewer pipelines (examples/extensions/subagent/README.md:24-31). Missing pieces to supply: budget/step limits (loop runs until model stops; `shouldStopAfterTurn` available in SDK only), sandbox, and PR etiquette (bash + gh).

## Strengths / weaknesses / fit

**Strengths**: cleanest embeddable loop of the cohort (injectable streamFn/convertToLlm/hooks); genuine tree-structured sessions with branch summarization; broad provider layer incl. subscription OAuth (Codex/Claude/Copilot) and local models; disciplined docs + evidence-of-thought design docs; honest security posture; strong supply-chain hygiene; session data published openly.
**Weaknesses**: zero default guardrails; fast-breaking APIs by policy; dominant-author governance with contributor gate; DIY cost for team-standard workflows; no MCP/ACP/LSP means ecosystem tools need bridging; experimental server layer not yet usable.
**Best fit**: expert individual developers who customize their harness; researchers/benchmarkers needing a scriptable, transparent, provider-agnostic loop; teams embedding an agent loop in their own product via SDK/RPC.
**Poor fit**: organizations wanting turnkey guardrails, approval workflows, SSO/policy controls, or API stability guarantees; users expecting MCP-ecosystem plug-and-play; unattended runs on untrusted code without separately-built sandboxing.
