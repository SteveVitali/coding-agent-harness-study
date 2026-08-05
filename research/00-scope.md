# 00 — Scope & Dashboard

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04.* The study's working dashboard, preserved as part of the record: pinned revisions, verification checklist, deviations from the protocol, classification corrections, and open questions as they stood at close (completion gate passed 2026-08-04). Follow-on work is enumerated in [12 §7](12-final-synthesis.md); the investigation reads end-to-end 00 → 01 → 02 → dossiers 03–08 → 09 → 10 → 11 → 12.

## Pinned revisions (cloned 2026-08-04, do not pull)

Local paths are the investigator's clones; upstream URLs and full 40-character SHAs are in the [README](../README.md).

| Repo | Local path | Pinned commit | Last commit date |
|---|---|---|---|
| sst/opencode (Anomaly) | `~/opencode` | `66fdd51f0d6d` | 2026-08-05 |
| block/goose | `~/goose` | `7e431ac6f804` | 2026-08-04 |
| OpenHands/OpenHands | `~/OpenHands` | `2c9ba42b19c7` | 2026-08-04 |
| OpenHands/software-agent-sdk | `~/openhands-software-agent-sdk` | `0c8f97aab8a2` | 2026-08-04 |
| badlogic/pi-mono | `~/pi-mono` | `588915ec7171` | 2026-08-04 |
| SWE-agent/mini-swe-agent | `~/mini-swe-agent` | `a83fcae82d2a` | 2026-07-22 |
| cline/cline | `~/cline` | `64993e78d562` | 2026-08-04 |

## Progress checklist

- [x] Repos cloned and pinned
- [x] Research scaffold created
- [x] Pass A+B+D+E dossiers (per-repo agents): all six done — opencode (03, 67 rows) / goose (04, 67 rows) / openhands+sdk (05, 56 rows) / pi-mono (06, 52 rows) / mini-swe-agent (07, 37 rows) / cline (08, 61 rows)
- [x] Citation spot-checks (≥3 per dossier), 26/26 passed: **mini-swe-agent 4/4** (default.py loop, litellm BASH_TOOL default, local.py sentinel, README "100 lines"); **openhands 5/5** (agent-canvas package.json, acp<0.11 pin, _extract_security_risk, llm_analyzer pass-through, exact-match str_replace); **opencode 4/4** (system.ts prompt fork, registry.ts apply_patch gate, websearch.ts A/B hash, prompt.ts runLoop); **pi-mono 4/4** (security.md no-sandbox stance, README rejected-features list, system-prompt.ts self-extensibility, agent types.ts injectable hooks); **goose 5/5** (Cargo.toml aaif-goose authorship, agent.rs thinking-block surgery comment, permission_judge.rs LLM read-only classifier, acp/provider.rs AcpProviderConfig, agent.rs invisible goal/grind nudges); **cline 4/4** (agent-runtime.ts 1,994-line loop + structured tool-call parts, MCP_PROTOCOL_VERSION "2024-11-05" stdio-only client, refs/cline/checkpoints private refs, ARCHITECTURE.md pointing at nonexistent files) — all verified 2026-08-04
- [x] 01-repository-inventory.md (draft-complete 2026-08-04)
- [x] 02-comparison-matrix.md (draft-complete 2026-08-04; 10 dimension groups × 6 subjects)
- [x] 09-interoperability.md (draft-complete 2026-08-04; 11 combos + layer diagram; MCP revisions resolved externally)
- [x] 10-benchmark-design.md (draft-complete 2026-08-04; all six invocation sections filled)
- [x] 11-evidence-ledger.md (merged from ledger-fragments/, 350 rows, 2026-08-04)
- [x] 12-final-synthesis.md (draft-complete 2026-08-04: exec synthesis, 20-question table, tradeoffs, Pass G frontier + coherence verdict, §5 portability verdict + adapter table, 7 audiences, gap analysis)
- [x] Completion gate (2026-08-04): prompt reread against deliverables (gap analysis at end of 12); 10 random ledger rows spot-checked — 9/10 exact, 1 line-number offset (openhands-app README quote at :10 not :36-41; claim true, citation corrected in fragment + merged ledger). ≤1 failure → no full audit per gate rule. Total citation checks this investigation: 36/36 substantively verified.

## Deviations from prompt

- Evidence ledger is written as per-repo fragments in `research/ledger-fragments/` during Phase 1 (parallel agents cannot safely share one file), merged into `11-evidence-ledger.md` afterward.

## External reference points (class: external, fetched 2026-08-04)

- MCP spec: latest revision **2026-07-28** (stateless core, removes Mcp-Session-Id, extensions framework; RC 2026-05-21). Most shipped clients likely still implement earlier revisions — compare each repo's handshake version string against this.
- ACP: Zed's Agent Client Protocol, JSON-RPC 2.0 over stdio, Apache-2.0; ACP Registry live since Jan 2026; adopted by Zed, JetBrains, and 25+ agents. Check which protocol version each repo's handshake negotiates.

## Classification corrections found so far

- OpenHands app repo (`~/OpenHands` @2c9ba42): prompt §0's "hosted/autonomous agent app: runtime, sandboxes, deployment" is **stale at the pin** — the repo is now **Agent Canvas** (`@openhands/agent-canvas` 1.9.0), a TS/React/Electron frontend with no Python engine; it consumes `openhands-agent-server` (pinned 1.39.1) from PyPI via uvx. The SDK repo is the sole authoritative engine; runtime/sandboxing lives in the SDK's agent-server + DockerWorkspace. The shipped CLI lives in a third repo (OpenHands-CLI, not cloned — out of pinned scope, note in synthesis). Drift evidence: app pins transitive `agent-client-protocol<0.11` because sdk 1.35.0 breaks against acp 0.11 (`config/defaults.json:13-14`).

- mini-swe-agent: prompt §0's "no tool-calling API" is **stale at the pin** — v2 default uses native litellm tool-calling with a single `bash` tool (`models/litellm_model.py:66-70`); the text-regex path survives as opt-in (`litellm_textbased_model.py`). "~100 lines" is 190 for the agent class (489 with env+model+run). Null-hypothesis framing for Pass G must say "single-bash-tool, no compaction, no harness features" rather than "no tool-calling".

- pi-mono: "minimal construction kit" needs the same nuance as mini-swe-agent — minimal *feature surface by authored philosophy* (README:496-506 explicitly rejects MCP/sub-agents/permission popups/plan mode/todos/background bash), but not a small codebase (~5,420 commits in 12 months, 409 test files, ~45 provider integrations, experimental CBOR remote-session layer). Zero enforced guardrails by default is documented intent (`packages/coding-agent/docs/security.md:31-37`); ~65% single-author commit share is a governance risk for the "build on it" decision.

- goose: governance correction confirmed and deeper than expected — pinned tree identifies as `github.com/aaif-goose/goose`, authors "AAIF <ai-oss-tools@block.xyz>" (Cargo.toml:12-15); Agentic AI Foundation @ Linux Foundation per README. Architectural correction: **no `goose-server` REST crate exists at the pin** — `goose acp` (stdio) and `goose serve` (ACP over HTTP+WS) are the only server surfaces; desktop app, npm TUI, and uniffi SDKs are ACP clients of the binary. Goose also runs other agents (Claude Code, Codex, Copilot, Amp, Pi) as *model providers* via ACP (`crates/goose/src/acp/provider.rs:50-68`).

- cline: prompt §0's "VS Code-first coding product; verify SDK claim" resolves as **SDK-first at the pin** — a Bun monorepo of published packages (`@cline/shared → llms → agents → core`, e.g. `@cline/core@0.0.69` on npm matching the repo, external check 2026-08-04) with VS Code, CLI/TUI (`cline` 3.0.49, zero VS Code imports), hub daemon, ACP agent, and a Tauri example all as hosts. The legacy XML `parseAssistantMessage`/`recursivelyMakeClineRequests` loop is gone; tools are Zod/JSON-schema structured. Checkpoints are no longer shadow-git: real-repo commits under `refs/cline/checkpoints/`, restore = `git reset --hard` + `git clean -fd` in a stash transaction. Two MCP clients coexist: SDK core hand-rolled stdio-only pinned `2024-11-05` vs extension McpHub on the official SDK (stdio/SSE/streamableHTTP) — headless MCP materially weaker than IDE MCP.

## Open questions

- Pass G/benchmark: since mini-swe-agent's default now uses tool-calling, should the control arm use the default (tool-calling) or the text-based model class for maximum-minimality? Provisional answer: default (it's what the project ships and benchmarks); note the option in 10-benchmark-design.
- Benchmark fairness (opencode): system prompt AND toolset fork on model-ID substrings (`system.ts:27-42`, `registry.ts:292-296`) — cross-harness runs partly measure opencode's per-model adaptations, not the harness loop. Must be a standing caveat in 10-benchmark-design §fairness.
- Benchmark T8/privacy (opencode): websearch A/B-routes to third-party Exa/Parallel MCP endpoints keyed on session-ID hash, keyless by default (`websearch.ts:30-37`) — relevant to the adversarial/exfil task design and to the security row in the matrix.
- ~~opencode/goose MCP spec-revision strings~~ RESOLVED 2026-08-04 (external, ledger §synthesis): both SDKs (@modelcontextprotocol/sdk 1.29.0, rmcp 3.0.0) negotiate latest revision **2025-11-25**; cline SDK-core client pinned **2024-11-05**; nobody ships 2026-07-28 as latest.
- Benchmark fairness (goose): four hidden LLM side-calls (two compaction paths, tool-pair summarization, session naming; plus permission judge in smart mode) and a per-turn `<turn-context>` injection spend tokens invisibly — token/cost metrics must come from the metering proxy, not harness self-reports; note which side-calls are configurable off in 10 §invocation.
