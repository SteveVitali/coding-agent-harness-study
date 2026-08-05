# 01 — Repository Inventory (Pass A cross-repo summary)

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commits in [00-scope](00-scope.md).* The condensed Pass-A cross-repo view; detail and citations live in dossiers 03–08.

## Cross-repo vitals (measured directly on pinned clones, 2026-08-04; class: commit-history/config, confidence: high)

| Repo | Core language | License | Commits 07-04→08-04 | Version at pin | Clone size |
|---|---|---|---|---|---|
| opencode | TypeScript (Bun, Effect 4β, AI SDK v6) | MIT | 533 | 1.18.13 | 571M |
| goose | Rust (+ Electron/React desktop, Node TUI) | Apache-2.0 | 273 | 1.45.0 | 1.3G |
| OpenHands app | TypeScript (React 19/Electron) | MIT | 261 | @openhands/agent-canvas 1.9.0 | 408M |
| openhands-sdk | Python ≥3.12 (uv workspace) | MIT | 142 | openhands-sdk 1.40.0 | 49M |
| pi-mono | TypeScript (Node ≥22, npm workspaces) | MIT | 589 | 0.83.0 lockstep | 79M |
| mini-swe-agent | Python | MIT | 8 | (frozen-cadence baseline) | 22M |
| cline | TypeScript (Bun monorepo, Vercel AI SDK) | Apache-2.0 | 411 | SDK 0.0.69 / CLI 3.0.49 / ext 4.1.3 | 482M |

Clone sizes include git history/deps — not a LoC proxy. All spot-check details: 00-scope §checklist (26/26 passed).

## Classification verdicts vs. prompt §0 (all with dossier citations)

| Subject | Provisional classification | Verdict |
|---|---|---|
| opencode | Productized terminal agent; TS/Bun rewrite; client/server split; embeddable server | **Confirmed, understated.** 33-package monorepo; one Effect-TS engine behind a Hono HTTP API; TUI/web/Electron/VS Code/Zed-ACP/Slack/GitHub Action are all thin clients. Repo moved to `anomalyco/opencode`; hosted console is in-tree but severable (03 §A). |
| goose | Rust + MCP/ACP composition; Linux Foundation | **Confirmed, deeper.** Now `aaif-goose/goose` (AAIF @ LF). The bespoke REST server is gone — ACP (stdio/HTTP+WS) *is* the API; desktop/TUI/SDKs are ACP clients. Also an ACP *client*: other coding agents (Claude Code, Codex, Copilot, Amp, Pi) are drivable as model providers (04 §A/F). |
| OpenHands app | Hosted/autonomous app: runtime, sandboxes, deployment | **Rejected at pin.** Repo is "Agent Canvas": a TS frontend with zero Python engine; installs `openhands-agent-server` from PyPI via uvx at launch. Runtime+sandboxing moved into the SDK repo's agent-server + workspace packages (05 §verdict). |
| openhands-sdk | V1 engine behind CLI/Cloud | **Confirmed.** Sole authoritative engine: agent loop, event-tree state, condenser, tools, MCP, ACP client, FastAPI agent-server, Docker/remote workspaces. Shipped CLI lives in a third repo (out of pinned scope) (05 §A/B). |
| pi-mono | Minimal, extensible TS harness kit | **Confirmed with amendment.** Minimal *feature surface by authored philosophy* (explicit "No MCP/sub-agents/permission popups/plan mode/todos/background bash" list) — not a small codebase (~5.4k commits/yr, 409 test files, ~45 providers, experimental CBOR server layer) (06 §A/D). |
| mini-swe-agent | ~100-line research baseline; bash-only; no tool-calling API | **Two corrections.** Agent class is 190 lines (489 with env+model+run); v2 default uses **native tool-calling** with a single `bash` tool (README/FAQ are stale; the repo's own migration doc admits it). Still the correct null-hypothesis baseline: single tool, linear history, no compaction, containment = environment class choice (07 §A/B). |
| cline | VS Code-first product; verify SDK claim | **SDK claim CONFIRMED REAL; "VS Code-first" stale.** Published `@cline/{shared,llms,agents,core}` (npm versions match repo), CLI 3.0.49 with zero VS Code imports, hub WS daemon, ACP agent; the VS Code extension is a ~90-file host adapter. Legacy XML loop deleted (08 §A/verdict). |

## One-paragraph architectural maps (details + citations in dossiers)

**opencode (03).** Single Effect-TS engine process exposes a Hono HTTP API + SSE; every UI is a generated-SDK client. Loop = `SessionPrompt.runLoop` folding normalized LLM events into immediately-persisted SQLite rows with per-step git snapshots. Tools host-exec with in-process wildcard permission rules (tree-sitter derives ask patterns); MCP client-only; ACP agent role v1; compaction = hidden "compaction" agent, non-destructive in DB. Commercial console/enterprise packages in-tree but optional.

**goose (04).** Rust workspace: `goose` core + CLI + providers + bundled MCP servers + uniffi SDKs. Loop = one ~1,000-line `try_stream!` in `agents/agent.rs` with steering, goal/grind nudges, stop-hook vetoes, 3 compaction mechanisms. Everything tool-shaped is MCP (client incl. builtin in-process servers); everything client-shaped is ACP (both roles). Sessions in SQLite with visibility-flip compaction. Permissions prompt-level (SmartApprove judge is an LLM call); Docker for extensions opt-in; built-in cron scheduler, recipes with retry success-checks, `goose review` check subagents.

**OpenHands = SDK engine + Canvas console (05).** SDK: `Agent.step()`/`LocalConversation.run()` over an event-sourced file tree (parent_id branching, flock), LiteLLM provider layer (pinned 1.93.0) with fallback+cache tiers, condenser (LLM summarizer, 240-event default), typed pydantic tools, real subagent LocalConversations, shell hooks with Stop-veto, confirmation policies fed by model-self-reported risk (or GraySwan API). Same engine serves as FastAPI/WS agent-server; DockerWorkspace runs the whole engine in a container (that is the sandbox). App repo: React/Electron console that drives N backends — OpenHands or any ACP agent — via a generated TS client; installs backend from PyPI at launch.

**pi-mono (06).** Four-layer kit: `pi-ai` (providers) ← `pi-agent-core` (792-line injectable loop: streamFn, convertToLlm, transformContext, before/afterToolCall, steering/follow-up) ← `pi-coding-agent` (CLI: 7 tools, extensions, skills, JSONL tree sessions, compaction) + `pi-tui`. Four run modes (interactive/print/JSON/RPC) + SDK. No MCP/ACP/LSP/permissions/sandbox by authored philosophy; extensions are TS modules with full permissions; OS/container isolation is the user's job.

**mini-swe-agent (07).** 190-line `DefaultAgent`: Jinja2 system+instance templates → litellm completion with one `bash` tool → parse → `subprocess`/`docker exec` per action (stateless shell) → observation template with fixed head/tail truncation → stdout-sentinel submission. Seven environment classes (local/docker/singularity/bubblewrap/contree/swerex_docker/swerex_modal) are the only containment. Batch runner = ThreadPoolExecutor emitting SWE-bench-official preds.json + trajectory JSON. Config entirely YAML.

**cline (08).** Layered published SDK: `shared → llms (~150 provider IDs / 9 vendor families over Vercel AI SDK) → agents (1,994-line stateless `AgentRuntime`, Zod-schema'd tools, ToolPolicy approvals) → core (SQLite sessions, agentic compaction with canonical transcript preserved, real-git checkpoint refs, cron automation, hub WS daemon, hand-rolled stdio MCP client @2024-11-05)`. Hosts inject executors: VS Code (terminal/diff-view), CLI/TUI (headless+yolo, ACP mode), hub dashboard, Tauri example. Legacy gRPC hostbridge survives for the JetBrains path.

## Cross-cutting Pass-A observations

- **Convergent architecture:** four of six (opencode, goose, OpenHands, cline) independently arrived at "one engine, many thin clients, protocol seam in between" — the seams differ (HTTP+SSE / ACP / REST+WS / npm packages+hub WS).
- **Nobody ships an OS-enforced sandbox for the default local path.** Containment everywhere = choose a container (OpenHands DockerWorkspace, mini-swe environment classes, goose opt-in extension containers) or bring your own (pi, opencode, cline).
- **Standards adoption is real but versions lag:** MCP clients negotiate 2025-11-25 at best (cline core: 2024-11-05); ACP present in 4/6 (opencode agent-role, goose both roles, OpenHands client-role, cline agent-role); AGENTS.md and SKILL.md (agentskills.io) supported by 5/6 — all except mini-swe-agent.
- **Docs lag code in every large repo** (cline ARCHITECTURE.md points at deleted files; mini-swe README misstates tool-calling; OpenHands classification churn; goose repo identity) — reinforcing evidence rule "implementation beats documentation."
