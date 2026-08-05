# 05 — OpenHands (app: OpenHands/OpenHands @ 2c9ba42b19c7; sdk: OpenHands/software-agent-sdk @ 0c8f97aab8a2)

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commits in the title.*

Follow-up work (traces not performed in this study): verify where `@openhands/typescript-client` and `openhands-automation` are generated/hosted (external repos); read agent-server conversation_service start-to-finish; sample one live `uvx openhands-agent-server` boot for startup latency; check OpenHands-CLI repo to confirm CLI = SDK consumer.

Citation prefixes: `app:` = ~/OpenHands, `sdk:` = ~/openhands-software-agent-sdk.

## Verdict: which repo is authoritative, what lives where

**The SDK repo is unambiguously the authoritative engine. The app repo contains no agent loop at all — legacy V0 is gone.** The provisional classification ("app = hosted product with runtime; SDK = engine behind it") is *outdated at this pin*: the app repo has been fully replaced by "Agent Canvas," a TypeScript/React/Electron frontend, npm package `@openhands/agent-canvas` v1.9.0 (app:package.json:2-5). A repo-wide search finds only four stray `.py` files (a UI tool script, a CI script, two e2e mock servers) — no Python engine. The app's own architecture doc states it is "not responsible for: Executing agent actions directly. Providing the sandbox or workspace isolation layer" (app:docs/architecture.md:14-19). The app's CLI (`bin/agent-canvas.mjs:5-11`) launches the Python backend **via `uvx` from PyPI** — packages `openhands-agent-server` (pinned 1.39.1) and `openhands-automation` (1.5.0) per app:config/defaults.json:3-8 — i.e., the app consumes the SDK repo's published artifacts; the dependency direction is strictly app → SDK, at package-registry (not source) level. The SDK README says the SDK is "the engine behind the OpenHands CLI and OpenHands Cloud" (sdk:README.md:44-46); the shipped CLI lives in a third repo (OpenHands-CLI — UNVERIFIED: not inspected, external). The app repo's git history still reaches back to 2024-03-13 (original OpenDevin/OpenHands repo re-used, tree replaced).

## A. Orientation

**SDK repo** — Python ≥3.12 (ruff targets py313), a uv workspace of four packages (sdk:pyproject.toml:2-3):
- `openhands-sdk` (v1.40.0, sdk:openhands-sdk/pyproject.toml:3): agent loop (`agent/`), conversation state machine + persistence (`conversation/`), LLM layer over LiteLLM (`llm/`), context/condenser/prompt registry (`context/`), events, MCP (fastmcp wrapper), skills, subagents, hooks, security analyzers, secrets, git helpers, workspace abstractions, plugin/marketplace/profile machinery, an ACP client agent (`agent/acp_agent.py`), critic (`critic/`).
- `openhands-tools`: terminal, file_editor, apply_patch, browser_use, glob, grep, task_tracker, planning_file_editor, delegate, preset (incl. per-model presets `gemini.py`, `gpt5.py`, `planning.py`, `subagents/`), gemini-specific tools, tom_consult, workflow.
- `openhands-agent-server`: FastAPI REST+WebSocket server exposing the engine (sdk:openhands-agent-server/openhands/agent_server/api.py:366-441) plus Docker image build (`docker/Dockerfile`, `build.py`), PyInstaller binary spec, VS Code service, desktop service.
- `openhands-workspace`: `DockerWorkspace` (spawns the agent-server container), `APIRemoteWorkspace` (All Hands runtime API), `CloudWorkspace` (app.all-hands.dev), apptainer variant.

**App repo** — TypeScript/React 19/React Router 7/Vite/Tailwind/Zustand + Electron desktop + helm chart + Docker image. `src/api/` holds service adapters (agent-server adapter, backend registry, ACP service, automation service, cloud); UI in `src/components|routes|stores`. Talks to backends through `@openhands/typescript-client` 1.36.1 npm dep (app:package.json) — a generated client for the agent-server REST API (generation site UNVERIFIED: not in either pinned repo; agent-server maintains a curated public OpenAPI document at sdk:agent_server/openapi.py:1-12). Deployment modes: `npx @openhands/agent-canvas` (local host processes via uvx), `--public` VM mode behind nginx (app:docs/SELF_HOSTING.md:1-52), Docker image (`docker/`), Kubernetes helm chart (`helm/agent-canvas/`), Electron desktop (`electron/main.mjs` spawns the same uvx stack). Automation backend (`openhands-automation` PyPI pkg, source repo not in scope) provides schedules/webhooks (Slack, GitHub, Linear per app:README.md:43-49).

**Licenses**: both MIT (app:LICENSE:1; sdk:LICENSE:1). **Cadence**: 278 app commits and 154 SDK commits since 2026-07-01 — very active; release-please in app, versioned PyPI/npm releases; SDK CI includes pyright, ruff, pre-commit, 665 test files (~4831 test functions under tests/sdk alone). **Cloud coupling**: default-off but hardcoded endpoints exist — `CloudWorkspace` targets `https://app.all-hands.dev` (sdk:openhands-workspace/openhands/workspace/cloud/workspace.py:57-81), `APIRemoteWorkspace` documents `runtime.all-hands.dev` with `runtime_class="sysbox-runc"` (sdk:remote_api/workspace.py:19-62); app ships a PostHog telemetry key in defaults.json.

**Architectural map**
```
app repo (TS)                         sdk repo (Python)
┌────────────────────────┐   REST/WS   ┌──────────────────────────────┐
│ Agent Canvas UI        │────────────▶│ openhands-agent-server       │
│ (React/Electron/helm)  │  typescript-│  FastAPI routers + sockets   │
│ bin/agent-canvas.mjs   │  client     │  ├ ConversationService       │
│  ├ spawns uvx agent-   │             │  └ wraps ↓                   │
│  │  server + automation│             │ openhands-sdk                │
│  └ backend registry:   │             │  Agent.step ⇄ LocalConversa- │
│    local/VM/docker/    │             │  tion.run ⇄ EventLog (files) │
│    OpenHands Cloud/ACP │             │  LLM(LiteLLM) · condenser ·  │
└────────────────────────┘             │  MCP · skills · hooks · sec  │
        also drives ACP agents         │ openhands-tools (terminal,   │
        (Claude Code, Codex, Gemini)   │  file_editor, browser, ...)  │
                                       │ openhands-workspace (docker/ │
                                       │  apptainer/cloud/runtime-API)│
                                       └──────────────────────────────┘
```

## B. One agent turn (authoritative engine = openhands-sdk, local mode)

1. **Input ingestion** — `LocalConversation.send_message()` wraps text into `Message(role="user")`, resets FINISHED/STUCK → IDLE, and consults `AgentContext.get_user_message_suffix()` to append trigger-activated skill content, recording `activated_knowledge_skills`; emits `MessageEvent` under the state lock (sdk:openhands-sdk/openhands/sdk/conversation/impl/local_conversation.py:1749-1812).
2. **System/instruction assembly** — `Agent.init_state()` emits a `SystemPromptEvent` whose static part is `AgentBase.static_system_message` (assembled by a `PromptRegistry` of `PromptSection`s — Role, ProblemSolving, CodeQuality, SecurityRiskAssessment, ModelSpecific, etc.) and whose `dynamic_context` (repo context/skills/secrets/datetime, incl. AGENTS.md-derived repo skills) is a second uncached block, explicitly to keep the static prefix prompt-cacheable across conversations (sdk:agent/agent.py:503-523; sdk:context/prompts/presets.py:1-11). Project instructions arrive as skills: `.openhands/skills/`, `AGENTS.md`, `~/.openhands/skills/`, legacy `~/.openhands/microagents/`, and the public OpenHands/extensions marketplace (sdk:context/agent_context.py:93-132). Two-tier `MEMORY.md` memory (user + project, 6000-char budget) also feeds the dynamic tier (sdk:context/memory.py:1-24).
3. **Context selection** — `Agent.step()` calls `prepare_llm_messages(state.view, condenser=self.condenser, llm=self.llm)` over an incrementally-maintained `View`; if the condenser decides to condense, a `Condensation` event is returned and the step ends without an LLM call (sdk:agent/agent.py:652-661).
4. **Request construction / provider abstraction** — `make_llm_completion()` → `LLM.completion()` → LiteLLM `completion`/`responses` (litellm pinned ==1.93.0, sdk:pyproject.toml:19). `LLM` is a large pydantic model (~3200 lines) handling base_url, model-feature gating (`model_features.py`), Anthropic `cache_control: ephemeral` markers (sdk:llm/message.py:200,384) but deliberately not for Gemini (sdk:llm/utils/model_features.py:141), retries (default 5, exp backoff 8–64s; sdk:llm/llm.py:326-329), and `FallbackStrategy` alternate-LLM failover (sdk:llm/fallback_strategy.py:39-79).
5. **Streaming** — opt-in; token deltas go to an `on_token` callback; without a callback it deliberately degrades to non-streaming (sdk:llm/llm.py:1499-1507).
6. **Tool-call parsing** — `classify_response()` splits TOOL_CALLS/CONTENT/REASONING_ONLY/EMPTY; `_get_action_event()` parses JSON args, `normalize_tool_call()` handles aliases/terminal fallback, `fix_malformed_tool_arguments()` repairs, pydantic-validates into a typed `Action`; malformed calls become `AgentErrorEvent`s fed back to the model (sdk:agent/agent.py:779-796,1158-1293).
7. **Authorization** — `_requires_user_confirmation()`: security analyzer scores pending actions (LLMSecurityAnalyzer = trusts the model's own `security_risk` argument, sdk:security/llm_analyzer.py:20-29; or GraySwan Cygnal external API), then `confirmation_policy` (AlwaysConfirm/NeverConfirm/ConfirmRisky, sdk:security/confirmation_policy.py:9-53) may set `WAITING_FOR_CONFIRMATION`; the next `run()` call executes pending actions as implicit approval (sdk:agent/agent.py:614-629,992-1033).
8. **Execution locus** — in local mode, tools run **in-process on the host**: `_execute_action_event()` invokes `tool(action, conversation)` on threads of a `ParallelToolExecutor` (sdk:agent/agent.py:1295-1354). Terminal = persistent tmux pane or subprocess PTY on the host (sdk:openhands-tools/openhands/tools/terminal/README.md:1-10). Isolation is opt-in by choosing a `DockerWorkspace`: then the *entire engine* runs inside a `ghcr.io/openhands/agent-server` container and the client becomes a `RemoteConversation` speaking httpx REST + a websockets event stream (sdk:openhands-workspace/docker/workspace.py:53-194; sdk:conversation/impl/remote_conversation.py:13-128). The boundary is the agent-server HTTP/WS API, authenticated by `x-session-api-key` (sdk:agent_server/sockets.py:105-166).
9. **Observation return** — executor returns an `Observation`; wrapped as `ObservationEvent` keyed to `action_id`/`tool_call_id`; batch results are emitted in original action order; hook-blocked actions become `UserRejectObservation` (sdk:agent/agent.py:301-318,1348-1354).
10. **Termination** — model calls `FinishTool`; trailing tool calls after it are discarded; state → `ConversationExecutionStatus.FINISHED` unless iterative-refinement critic injects a follow-up user message or a Stop hook denies finishing and injects feedback (sdk:agent/agent.py:200-226,320-351; sdk:conversation/impl/local_conversation.py:1885-1913).
11. **Persistence** — every event is a JSON file in a file-backed `EventLog` with flock locking and an id↔index map; events carry `parent_id`, forming a **tree** with `path_to_root()`/`active_branch()` — i.e., first-class history branching, legacy events fall back to a linear chain (sdk:conversation/event_store.py:30-116). Token/cost history persists in `ConversationStats` (sdk:conversation/conversation_stats.py:13-28).
12. **Compaction** — `LLMSummarizingCondenser` (max_size=240 events, keep_first=2): middle events are "forgotten" and replaced with an LLM-written summary event; context-window overflow and provider "malformed history" errors both route into `CondensationRequest` recovery (sdk:context/condenser/llm_summarizing_condenser.py:38-53,229-270; sdk:agent/agent.py:732-773). What's lost: raw event content of the forgotten span; what's kept: first 2 events, a summary, and the tail.
13. **Failure/retry/cancellation/timeout** — outer `run()` loop caps `max_iteration_per_run=500` and a max-budget check (`MaxBudgetReached` error event); `StuckDetector` scans the last 20 events for 5 repetition patterns and first *nudges* the model once on an action-error streak before hard STUCK (sdk:conversation/impl/local_conversation.py:204,654-677,1954-1984; sdk:conversation/stuck_detector.py:22-48). Cancellation via a `CancellationToken` threaded into tool batches; pause via lock-acquired status flip.

## C. Matrix inputs

| Dimension | Label | Note (citation) |
|---|---|---|
| TUI/CLI | external | Shipped CLI is a separate repo (OpenHands-CLI) built on the SDK (sdk:README.md:44-46). App's `agent-canvas` CLI only launches the web stack (app:bin/agent-canvas.mjs:5-11). |
| IDE integration | partial | Agent-server bundles a VS Code service/extensions + vscode_router (sdk:agent_server dir); no editor plugin in either repo. |
| Web UI | native | The entire app repo is the web UI; also Electron desktop (app:electron/main.mjs). |
| Headless/CI | native | `examples/01_standalone_sdk` scripts + `examples/03_github_workflows` (sdk:examples listing); Conversation.run() is a plain Python call. |
| Server API | native | FastAPI REST + WS, ~20 routers incl. bash/git/files/skills/sub_agents/hooks/mcp (sdk:agent_server/api.py:403-441); curated public OpenAPI doc (sdk:agent_server/openapi.py:1-12). |
| SDK quality | native/high | Typed pydantic models throughout, py.typed, 665 test files, 5 example families, per-module AGENTS.md docs; docs site external (docs.openhands.dev). Sharp edges: giant classes (llm.py 3197 lines, local_conversation.py 2976). |
| Multi-session | native | Agent-server ConversationService manages many conversations; app UI multiplexes backends+conversations (app:docs/architecture.md:21-24). |
| Session branching | native | Event tree with parent_id + `path_to_root`/`active_branch` (sdk:conversation/event_store.py:91-116). |
| Approval/confirmation UX | native | Risk-conditional policies (ConfirmRisky) + WAITING_FOR_CONFIRMATION status; implicit approval on next run() (sdk:agent/agent.py:614-629,1027-1031). |
| Diff/checkpoint UX | partial | git_router + sdk git module + UI git-service; no automatic per-step checkpointing found (absence — inference). |
| Provider abstraction & leaks | native w/ leaks | LiteLLM everywhere; field names leak it: `litellm_extra_body`, `custom_llm_provider`, litellm exceptions in agent code (sdk:llm/llm.py:577; agent.py imports litellm-derived exception types via sdk wrappers). |
| OpenAI-compatible + local | native | `base_url`, `ollama_base_url` fields (sdk:llm/llm.py:271,436). |
| Model-specific adaptations | native | `model_features.py` registry + `ModelSpecificSection` prompt section + per-model tool presets gemini/gpt5 (sdk:llm/utils/model_features.py:51-140; tools/preset/). |
| Reasoning models | native | responses-API path (`litellm_responses`), ThinkingBlock/RedactedThinkingBlock/ReasoningItemModel preserved in ActionEvents (sdk:agent/agent.py:60-63,1271-1274). |
| Prompt caching | native | cache_control ephemeral + static/dynamic system-prompt tiers designed for cross-conversation cache hits (sdk:agent/agent.py:503-510; llm/message.py:200,384). |
| Token/cost accounting | native | ConversationStats persists costs/latencies/token_usages; budget enforcement in run() (sdk:conversation/conversation_stats.py:13-28; local_conversation.py:1954-1960). |
| Retry/fallback | native | 5 retries exp backoff + FallbackStrategy list of alternate LLMs (sdk:llm/llm.py:326-329; llm/fallback_strategy.py:39-79). |
| Drive another complete agent | native | **ACPAgent** runs Claude Code/Gemini CLI/Codex as the whole engine via ACP subprocess (sdk:agent/acp_agent.py:1-16); Delegate tool spawns sub-LocalConversations (sdk:tools/delegate/impl.py:66-82,230). |
| Built-in tools | native | terminal, file_editor, apply_patch, browser_use, glob, grep, task_tracker, planning_file_editor, delegate, vision_inspect, think/finish builtins (sdk:openhands-tools listing; sdk/tool/builtins). |
| Tool registration/schemas | native | `ToolDefinition` with pydantic `action_type` → OpenAI schema at completion time (sdk:agent/agent.py:514-518). |
| MCP | native (client) | Thin sync wrapper over `fastmcp.Client`, fastmcp>=3.0.0 (sdk:mcp/client.py:1-27; sdk:openhands-sdk/pyproject.toml:11). No spec-revision string in repo — version negotiation delegated to fastmcp (absence). Server-side: agent-server has mcp_router + OAuth store. |
| ACP | native (both directions) | SDK consumes ACP agents (acp_agent.py); app UI drives "any ACP-compatible agent" (app:README.md:9-12) with a mock ACP server in e2e tests. |
| LSP | absent | No LSP integration found (grep across both repos; only incidental matches). |
| Browser tool | native | browser-use library wrapped as MCP server (sdk:tools/browser_use/impl.py:1). |
| Shell execution | native | Persistent tmux/subprocess-PTY session, soft timeouts, reset (sdk:tools/terminal/README.md:1-12). |
| File-edit mechanism | native | view/create/str_replace/insert/undo_edit; exact literal match with strip-retry, unique-match enforced, **no fuzzy matching**; optional linting (sdk:tools/file_editor/definition.py:26-57; editor.py:200-240). Separate OpenAI-style apply_patch tool. |
| Git integration | native | sdk git module, git_router in server, PR guidance prompt section (`PullRequestsSection`, sdk:context/prompts/presets.py imports). |
| Custom-tool API | native | Subclass ToolDefinition/ToolExecutor; register in agent `tools` list (sdk:agent docstring agent.py:376-384). |
| Tool-result truncation | native | `DEFAULT_TEXT_CONTENT_LIMIT` truncation at serialization (sdk:llm/message.py:336,471-479). |
| Repo maps | absent | No repomap/repo_map implementation (grep). Repo context comes from skills/AGENTS.md instead. |
| Search/retrieval | partial | glob + grep tools; no embedding/index retrieval found (absence). |
| Agent Skills | native | AgentSkills SKILL.md standard w/ frontmatter, triggers, resources; marketplace + public extensions repo (sdk:skills/skill.py:178-230). |
| AGENTS.md/microagents | native | AGENTS.md → project skill; `~/.openhands/microagents/` legacy path still loaded (sdk:context/agent_context.py:93-132). |
| Memory | native | Two-tier MEMORY.md, agent-maintained, 6000-char budget (sdk:context/memory.py:1-24). |
| Compaction/condenser | native | Pluggable CondenserBase; LLMSummarizingCondenser default recipe; pipeline condenser (sdk:context/condenser/). |
| Plan modes | native | Planning prompt preset + planning_file_editor + PlanningSection (sdk:context/prompts/presets.py:6-9). |
| Todos | native | task_tracker tool in default preset (sdk:tools/preset/default.py:55). |
| Subagents/delegation | native | Markdown-frontmatter agent defs (model/tools/skills/mcp/permission_mode/condenser per agent) — real isolated LocalConversations, not prompt games (sdk:subagent/schema.py:27-49; tools/delegate/impl.py:230). |
| Parallelism | native | ParallelToolExecutor per turn; delegate spawn of multiple agents; server hosts many conversations (sdk:agent/agent.py:395-403). |
| Background execution | native | Automation backend (schedules/webhooks) in app stack; bash_service/bash events in server (app:config/defaults.json; sdk:agent_server/bash_service.py). |
| Scheduling | via-extension | `openhands-automation` package (separate, uvx-installed) + app UI automations (app:README.md:43-49). |
| Exec environments | native | Host exec (default local), DockerWorkspace (container w/ agent-server image), Apptainer, APIRemoteWorkspace (All Hands runtime API, sysbox-runc), CloudWorkspace, helm/k8s for the app (sdk:openhands-workspace listing; app:helm/). Image build: multi-stage Dockerfile + PyInstaller binary that can be COPY'd onto arbitrary bases incl. SWE-bench images (sdk:agent_server/docker/Dockerfile:24,156-203). |
| FS/network isolation | partial | Only when you pick a container/remote workspace; local mode has none — self-hosting doc warns the agent "can read and write the filesystem... execute shell commands, and reach the network" (app:docs/SELF_HOSTING.md:6-11). |
| Credential/secret handling | native | SecretRegistry injected into prompts w/ descriptions; secret redaction in server; ACP credential lifecycle files for Codex/Gemini OAuth (sdk:agent/agent.py:508-511; agent_server/_secret_redaction.py; agent/acp_file_credentials.py). |
| Multi-user architecture | partial | Agent-server auth = shared session API keys, not per-user identity (sdk:agent_server/config.py:216-227); app public mode = one pasted key (app:docs/SELF_HOSTING.md:50-52). Org/multi-tenant is OpenHands Cloud's job. |
| Permission model | mixed — see below | Confirmation mode is a **policy prompt the model feeds** (risk is model-self-reported under LLMSecurityAnalyzer); the OS-enforced boundary exists only if you deploy DockerWorkspace/remote runtime. |
| Prompt-injection defenses | partial | LLM self-assessment + optional GraySwan Cygnal API analyzer + `defense_in_depth` policy rails + toolshield helpers (sdk:security/ listing; grayswan/analyzer.py:28-40). None on by default beyond the security_risk prompt section. |
| Verification loop | native | CriticMixin evaluates actions; iterative-refinement can bounce a Finish back with follow-up; Stop hooks can deny finishing (sdk:agent/agent.py:1283-1291,320-351). |
| Evals in-repo | partial | Real-LLM integration (`t*`), behavior (`b*`, LLM-judge), condenser stress (`c*`) tests with success-rate reporting (sdk:tests/integration/README.md:1-17). **No SWE-bench harness in-repo** (Dockerfile only accommodates SWE-bench base images); README's 77.6 SWE-bench badge links a Google Sheet — methodology (model, budget, retries) not reproducible from this repo. |
| Hooks/events | native | Command (shell) and agent hooks: PreToolUse-style action blocking, UserPromptSubmit blocking, Stop veto w/ feedback (sdk:hooks/config.py:42-58; agent.py:632-644). Everything is an event stream with callbacks. |
| Config portability | partial | Agent/LLM/profile config are pydantic JSON; app stores backend registry in localStorage; profiles/llm_profile_store in SDK. |
| Setup cost | medium | `uvx`/`npx` one-liners for local; Docker/helm for isolation; container startup + image pull for sandboxed runs. |
| Effort: add tool | low | one ToolDefinition subclass. Add provider: usually zero-code (LiteLLM name) + optional model_features entry. Change loop: subclass AgentBase (ACPAgent proves swappability). Embed: `pip install openhands-sdk` + 10-line Conversation script (sdk:README quickstart). |

## D. Governing philosophy

An **engine/console split with a protocol seam**. The engine (SDK) owns everything semantically hard: typed event-sourced state, condensation, risk policy, retries/fallback, tool schemas, sub-agents — and exposes it as both a Python API and a REST/WS server; the console (app) is deliberately dumb ("not responsible for executing agent actions", app:docs/architecture.md:14-19) and treats even OpenHands itself as just one of several ACP-speaking backends (app:README.md:9-12). Complexities owned: context economy (cache-tiered prompts, condenser, truncation — sdk:agent/agent.py:503-510), robustness (malformed-tool-call repair, content-filter nudges, stuck nudging), model heterogeneity (model_features + per-model presets). Complexities pushed onto users/infra: **actual isolation** (choose your workspace; local default is unsandboxed), multi-user auth (shared API keys, sdk:agent_server/config.py:216-227), TLS/nginx, and eval reproducibility. The bet: standardize the *protocol surface* (ACP, MCP, SKILL.md, AGENTS.md, agent-server OpenAPI) so both engines and frontends are swappable.

## E. Hidden tradeoffs

- **CONFIRMED — app↔SDK version drift is a live failure mode**: the app pins `agent-client-protocol<0.11` because "openhands-sdk 1.35.0 requires agent-client-protocol>=0.10.1 with no upper bound, but acp 0.11.0 reordered the ACP prompt() arguments and breaks the SDK's ACP client" (app:config/defaults.json:13-14). Registry-level coupling (uvx at launch) means the UI must carry compatibility shims (`minimumAgentServer: 1.28.0`).
- **CONFIRMED — confirmation mode ≠ sandbox**: default risk signal is the model's own `security_risk` argument (sdk:security/llm_analyzer.py:20-29), popped from tool args the model writes (sdk:agent/agent.py:1041). A prompt-injected model can label everything LOW; with `ConfirmRisky` that routes around the human. OS enforcement exists only via Docker/remote workspaces; local default is host exec (app:docs/SELF_HOSTING.md:6-11 warns exactly this).
- **CONFIRMED — LiteLLM leakage**: `litellm_extra_body`, litellm exception taxonomy, and a hard `litellm==1.93.0` pin with a dated `exclude-newer-package` escape hatch (sdk:pyproject.toml:8-19); agent behavior (vLLM token events) keys off litellm raw response fields (sdk:agent/agent.py:1356-1369).
- **CONFIRMED — event-stream growth is a designed-around problem**: code comments admit "conversations can reach 30k+ events" and init deliberately avoids materializing history (sdk:agent/agent.py:444-463); every event is a separate file with flock — NFS explicitly unsupported (sdk:conversation/event_store.py:36-40).
- **CONFIRMED — lossy condensation**: forgotten span survives only as one LLM summary; `keep_first=2` + tail (sdk:context/condenser/llm_summarizing_condenser.py:229-270). Malformed-history provider errors are *also* solved by condensing (agent.py:732-751), which can paper over event-stream bugs (the code itself warns this).
- **CONFIRMED — cloud endpoints baked in**: app.all-hands.dev / runtime.all-hands.dev defaults in openhands-workspace (E-cited above); PostHog key ships in app defaults.json.
- **RISK — multi-user tax**: agent-server auth is a flat list of session API keys; anyone with the key gets full bash/file/git routers. Mechanism: no per-user principal in Config (sdk:agent_server/config.py:216-227); team isolation requires one server per trust domain.
- **RISK — giant-module maintainability**: llm.py 3,197 lines / local_conversation.py 2,976 / agent.py 1,446 with duplicated sync+async bodies (step/astep are near-clones, agent.py:613-990); divergence between the twins is a standing bug source. Mitigated by unusually heavy tests (665 files).
- **RISK — uvx-at-launch supply chain & latency**: the app installs the Python backend from PyPI at startup (app:bin/agent-canvas.mjs:5-11); first run downloads an entire Python stack; offline/pinned-env installs need extra work.

## F. Interop surfaces

- **MCP client**: `MCPClient(fastmcp.Client)` (sdk:openhands-sdk/openhands/sdk/mcp/client.py:24-60); handshake/spec revision delegated to fastmcp>=3.0.0 (sdk:openhands-sdk/pyproject.toml:11) — no revision string in-repo. Server side: `mcp_router` + `mcp_oauth_store` in agent-server (dir listing).
- **ACP**: full client implementation `ACPAgent` using `acp.client.connection.ClientSideConnection` over stdio subprocess, incl. permission-request relay and MCP server pass-through (sdk:agent/acp_agent.py:1-53); app e2e ships a mock ACP server (app:tests/e2e/mock-llm/scripts/mock-acp-server.py).
- **Driving OpenHands from outside**: REST `/api/...` + WS `/sockets/...` with `x-session-api-key` auth (sdk:agent_server/sockets.py:105-166); curated OpenAPI document (sdk:agent_server/openapi.py); `RemoteConversation` is the reference client (httpx + websockets, remote_conversation.py:13-128).
- **Another harness inside its sandbox**: two supported routes — (1) the agent-server image best-effort installs Node 22 so ACP agents (Claude Code/Codex/Gemini CLI) run *inside* the sandbox container (sdk:agent_server/docker/Dockerfile:156-203); (2) bash_router/exec gives arbitrary process execution in the workspace.
- **External orchestration of workers**: each DockerWorkspace is an independently addressable agent-server (host port 30000-39999, workspace.py:186-194); the app's backend-registry pattern shows N servers driven by one client; sub_agents_router exposes delegation over REST.

## G. Benchmark inputs

- **Headless**: pure-Python `Conversation(agent, workspace).send_message(...); conversation.run()` (sdk:README quickstart); sandboxed variant swaps in `DockerWorkspace(server_image=...)` — same code path (examples/02_remote_agent_server).
- **Fixed endpoint injection**: `LLM(model=..., base_url=..., api_key=...)`; any OpenAI-compatible endpoint via LiteLLM naming; `ollama_base_url` for local (sdk:llm/llm.py:271,436).
- **Scripting approvals**: `NeverConfirm` policy = fully unattended (sdk:security/confirmation_policy.py:35-41); no analyzer configured → risks UNKNOWN → no prompts unless AlwaysConfirm.
- **Budget/limit controls**: `max_iteration_per_run` (default 500) and max-budget both first-class (local_conversation.py:204,1954-1960); stuck detector can be disabled via `stuck_detection=False` (local_conversation.py:205).
- **Fairness blockers**: Docker mode pays image pull + container + server boot before the first token; uvx app path pays PyPI resolution; stuck-detector nudges inject extra user messages (changes transcript vs other harnesses); default condenser at 240 events alters long-run context vs harnesses that fail hard. In-repo integration tests are completion/behavior checks with an LLM judge, not a controlled benchmark (sdk:tests/integration/README.md) — model/budget/retry controls for the advertised SWE-bench 77.6 are not in this repo (external sheet).

## H. Portability-adapter inputs

- **Native mechanisms an adapter would use**: (1) **Skills** — AgentSkills SKILL.md with frontmatter parsing via `python-frontmatter` (sdk:skills/skill.py:13,178-230) for procedures; (2) **AGENTS.md** — auto-loaded as always-active repo context skill (sdk:context/agent_context.py:127-132) for policy; (3) **files/git** — workspace is a plain directory + git module; (4) **MCP** — `mcp_config` on Agent or per-subagent frontmatter (sdk:subagent/schema.py:38-39); (5) **hooks** — shell-command hooks for lifecycle enforcement (sdk:hooks/config.py:42-58); (6) subagent .md files map 1:1 to role-agents with own model/tools/permission_mode.
- **What it would silently fight**: LLMSummarizingCondenser discarding mid-task state after ~240 events (plan files on disk survive; in-context task lists don't — task_tracker mitigates); stuck detector nudging/halting legitimate retry loops (repeated test-run/fix cycles can look like action-error streaks, stuck_detector.py patterns 1-2); confirmation mode blocking unattended runs unless `NeverConfirm` is set; skills auto-injection competing with adapter-supplied prompts for the dynamic tier; memory loader injecting `MEMORY.md` you didn't write.
- **Unattended spec→plan→tests→implement→review→verify→PR**: high coverage. Plan preset + planning_file_editor + task_tracker (plan), terminal+file_editor (tests/implement), CriticMixin/iterative refinement + Stop hooks (review/verify gates), git module + PullRequestsSection prompt guidance (PR). All drivable headless with NeverConfirm inside DockerWorkspace. Missing pieces: no built-in PR-creation tool verified in-repo (terminal `gh` instead — inference), and review depends on configuring critic/hooks rather than a shipped reviewer.

## Strengths / weaknesses / fit

**Strengths**: cleanest engine/UI separation in the study (protocol seam: OpenAPI+ACP+MCP+SKILL.md); event-sourced state with real branching; production-grade LLM layer (caching tiers, fallback, model features); subagents/hooks/skills adopt the emerging cross-harness standards; sandbox = same engine in a container (no separate "runtime" codebase anymore); exceptional test volume.
**Weaknesses**: security story defaults to model-self-reported risk; local default unsandboxed; registry-level app↔SDK coupling drifts; LiteLLM pin fragility; sync/async code duplication; docs and eval methodology live outside the pinned repos; CLI outside both repos.
**Best fit**: teams embedding a code agent in their own product/infra; self-hosted always-on agent fleets (VM/k8s); running third-party agents (Claude Code/Codex) under one console; benchmark rigs needing container-isolated workers with a REST control plane.
**Poor fit**: single-binary local CLI use (that's a different repo); zero-Docker environments wanting strong isolation; multi-tenant SaaS without building auth on top; anyone needing an in-repo, reproducible SWE-bench harness.
