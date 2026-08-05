# 12 — Final Synthesis

*Part of the [Coding-Agent Harness Study](../README.md) · snapshot 2026-08-04 · citations pinned to the commits in [00-scope](00-scope.md).* Benchmark execution (10) is the follow-on project; unresolved items are enumerated in §7.

Evidence discipline: every claim here traces to dossiers 03–08 and ledger 11 (350 merged rows + synthesis additions; 26/26 dossier spot-checks passed). Claims that static reading cannot prove are marked **hypothesis → experiment** and routed to 10-benchmark-design.

## 1. Executive synthesis (which layer each project occupies)

Six projects, four architectural species:

- **opencode** — a *product platform*: one heavily-engineered TS engine (SQLite state, git snapshots, permission rules, compaction) behind an HTTP API, with seven first-class UIs. It occupies the **loop + client layers** and owns more model-quirk complexity than anyone (per-family prompts/toolsets, patched provider SDKs, fuzzy edit repair, LSP feedback).
- **goose** — a *protocol broker*: the only subject implementing MCP both roles **and** ACP both roles, able to run competitors' agents as its "model." It occupies the **loop + workflow layers** (recipes, scheduler, review checks) and bets everything on standards composition — including replacing its own REST API with ACP.
- **OpenHands (SDK + Canvas)** — an *engine/runtime/console stack*: the only subject where the sandbox IS the deployment unit (whole engine in a container behind REST/WS), with event-tree state and the richest standards adoption (skills w/ triggers, subagent md, hooks w/ Stop-veto, memory). It occupies **loop + runtime + server layers**; its own UI treats OpenHands as just one ACP backend among competitors.
- **pi-mono** — a *construction kit*: the cleanest injectable loop (streamFn/convertToLlm/hooks), tree sessions, ~45 providers, and an authored refusal to own MCP/permissions/subagents/plan/todos. Pure **loop layer**, by design.
- **mini-swe-agent** — the *control arm*: 190 readable lines, one bash tool, per-action subprocess, YAML-only config, SWE-bench-official output. **Loop layer at minimum viable size**, plus the study's only first-class benchmark harness.
- **cline** — an *engine-as-product inversion*: the old VS Code extension is now a host adapter over published npm packages (stateless AgentRuntime + stateful core), with a headless CLI, hub daemon, ACP mode, and cron automation. **Loop + client layers**, converging on the same shape as opencode from the IDE side.

**Convergences (independent, therefore load-bearing):** every large subject split engine from UI behind a seam (HTTP/ACP/npm/REST); every subject defaults to **host execution with prompt-level authority** — OS enforcement exists only where a container is explicitly chosen; 5/6 adopted the same three file conventions (AGENTS.md, SKILL.md, mcpServers JSON); all four big loops converged on: native tool-calling, exact-match-first editing, LLM-summarizing compaction, per-step persistence, real subagents.

**Divergences that matter:** where model-quirk handling lives (opencode/goose: in the core loop — leaky by their own comments; OpenHands/cline/pi: in provider registries — contained); state shape (linear log vs tree — OpenHands/pi/opencode have real branching); and who may drive whom (goose and OpenHands drive others; opencode and cline can only be driven; pi and mini-swe abstain).

## 2. The twenty questions

| # | Question | Verdict (one line) | Conf. | Support |
|---|---|---|---|---|
| 1 | Best daily coding agent | opencode — deepest interactive surface (sessions/forking/revert/LSP/background subagents) with serious loop tests | med-high | 03 §C/E; ledger 03 |
| 2 | Strongest general-purpose shell | goose — MCP×2 + ACP×2 + recipes + scheduler + gateway; "agent of agents" is shipped, not roadmap | high | 04 §C/F |
| 3 | Best production runtime for hosted coding agents | OpenHands SDK + agent-server — container-as-engine, REST/WS control plane, event-sourced state, helm path | high | 05 §B/C/F |
| 4 | Best foundation for a custom harness | pi-agent-core for loop-level control (injectable seams, no fork); cline SDK if you want batteries + npm; OpenHands SDK if Python/infra | med | 06 §C; 08 §A; 05 §C |
| 5 | Best experimental baseline | mini-swe-agent — by design and by output format; caveat: default is now tool-calling | high | 07 §B/G |
| 6 | Strongest context engineering | OpenHands — condenser + 2-tier cached system prompt + triggered skills + MEMORY.md; opencode close second (compaction+prune+LSP) | med | 05 §B/C; 03 §B |
| 7 | Strongest security/sandboxing | OpenHands (interactive/hosted: engine-in-container); mini-swe (batch: bubblewrap/docker per task). No subject sandboxes the default local path | high | 05 §B.8; 07 §C; matrix §7 |
| 8 | Strongest extension model | goose — everything is a protocol (any MCP server, any ACP agent, skills, recipes); opencode's plugin API is richer in-process but `experimental.*`-unstable | med-high | 04 §C; 03 §E |
| 9 | Cleanest internal architecture | pi (layering + injectable loop); cline SDK second (shared→llms→agents→core); the two biggest loops are monoliths (goose 1,000-line stream; opencode Effect complexity) | med | 06 §A/B; 08 §A; 04 §E |
| 10 | Most dangerous hidden assumptions | Model-self-assessed safety: OpenHands' default risk = the model's own `security_risk` arg; goose's SmartApprove judge is an LLM call; cline SDK auto-approves unlisted tools; opencode forks behavior on model-ID substrings | high | 05 §E; 04 §E; 08 §E; 03 §E |
| 11 | Most harness overhead | goose — up to 4 hidden LLM side-calls + per-turn moim injection (one not user-disableable); hypothesis → measure in T1/T7 | med | 04 §G; 10 §goose |
| 12 | Genuinely portable components | SKILL.md skills, AGENTS.md, mcpServers JSON shape, plain files+git state, mini-swe trajectory format | high | 09 §6-8,10 |
| 13 | Technically coherent combinations | goose→opencode via ACP; opencode inside OpenHands sandbox; Canvas over any ACP agent; skills/AGENTS.md shared everywhere; mini-swe as control | high | 09 §1,3,6,7,10 |
| 14 | Redundant/overengineered combos | goose⇄OpenHands orchestration today (wrong-direction ACP roles, double transcripts); any double-harness stack where one engine + a container suffices | med | 09 §2 |
| 15 | Unified best-of-all-worlds feasible? | As one codebase: no (loop-layer philosophies conflict). As a composition over the file/protocol layer: yes, and 5/6 already interoperate there | med | 09 §11; §5 below |
| 16 | What would it look like | Stateless injectable loop (pi/cline shape) + OpenHands-style workspace isolation + goose-style protocol brokerage + skills/AGENTS.md/files/tests as the portable methodology; see §4 frontier | med | §4 |
| 17 | Missing from all six | (a) OS-enforced default-on isolation for local runs; (b) mechanical "requirement demonstrated" verification (all six equate exit-0 or model assertion with success); (c) blocking (not advisory) prompt-injection defense | high | matrix §7/§8 |
| 18 | Most likely to stay strategically relevant | OpenHands SDK (isolation + control plane + standards grow in value as models absorb loop cleverness); goose's broker position second | med | §4 |
| 19 | Benefits most from strong models | mini-swe-agent (harness does nothing → all gains accrue) and pi (no guardrails to constrain a strong model); hypothesis → T1-T7 vs control arm | med (routed) | 10 §fairness |
| 20 | Adds most value beyond the model | opencode (edit-repair cascade, LSP feedback, invalid-tool routing, retry taxonomy compensate weak models); hypothesis → run weak-model arm, T1/T3/T5 | med (routed) | 03 §B/E; 10 |

Undecidable-from-static-analysis note: #11, #19, #20 are explicitly hypotheses; the deciding experiments are specified in 10-benchmark-design (controlled model, metering proxy, mini-swe as null hypothesis).

## 3. Tradeoff analysis

- **Minimality ↔ safety.** The two poles are pi/mini-swe (zero default guardrails, honestly documented) and OpenHands (policy machinery + optional container). The study's sharpest finding: *midpoint* safety — permission prompts without OS enforcement (opencode, goose, cline) — costs UX and tokens while a routed-around boundary provides mostly the *feeling* of safety; all three admit in code or docs that an approved shell command is unconstrained.
- **Ergonomics ↔ embeddability.** opencode maximizes ergonomics and pays with a fork-to-modify loop; pi maximizes embeddability and pays with DIY everything; cline and OpenHands genuinely hold both (published SDK + product UI) at the price of pre-1.0 churn (cline) and giant modules (OpenHands).
- **Generic protocols ↔ deep integration.** goose shows the protocol-maximal ceiling: ACP-as-API forces product features into `_meta` extensions; opencode shows the integration ceiling: 16 patched dependencies and model-family forks in core. Neither cost is visible in feature lists.
- **Model freedom ↔ model-specific tuning.** Every provider abstraction leaks; the honest designs *declare* the leak (pi compat flags, OpenHands model_features registry) while the risky ones *hide* it (opencode substring forks; goose in-loop surgery). Benchmark consequence: harness comparisons are partially model-adaptation comparisons (standing caveat in 10).
- **Autonomy ↔ approval.** Approval systems converge to a bypass in unattended mode (yolo/auto/NeverConfirm), and cline shows the end state: a *separate system prompt* for unattended runs. The real control surface for autonomy is not approvals but budgets + gates (goose retry-checks, OpenHands Stop-hooks/critic).
- **Remote isolation ↔ local responsiveness.** Only OpenHands offers true remote isolation, and pays in boot latency and a REST/WS hop; everyone else keeps latency and gives up isolation. Nobody has local *and* isolated (e.g., seatbelt/landlock) — a genuine open slot.
- **Declarative ↔ programmable.** Declarative wins portability (mini-swe YAML, goose recipes, cline cron md) but the formats are harness-locked except mini-swe's; programmable wins power (pi extensions, opencode plugins) but is TS-in-process and unstable. The portable middle is *files + skills*, which is §5's argument.
- **Subagent specialization ↔ token overhead.** All big four ship *real* subagents (isolated contexts), which multiplies transcripts and system prompts per delegate; none measures or reports the overhead. Hypothesis → T3/T7 with subagents on/off.

## 4. Pass G — complexity vs. demonstrated benefit, and the frontier

Using mini-swe-agent as the null hypothesis (190 lines, one tool, no compaction — >74% SWE-bench Verified claimed by its README, not re-verified here):

- **Plausibly improves task success** (hypotheses → 10): fuzzy edit repair + LSP feedback (opencode; T3/T5), condenser/compaction for long tasks (all big four; T7), malformed-call repair (OpenHands/opencode; weak-model arm), stuck/doom-loop detection (T7 rework metric).
- **Clearly improves usability, not necessarily success:** session trees/forking, checkpoints/revert, TUIs, hub/multi-session — product surface, valuable, but orthogonal to completion.
- **Improves safety only when OS-backed:** container workspaces (OpenHands, mini-swe envs). Prompt-level permissions and LLM judges are **unproven-by-construction** (the enforcement point can be argued with); T8/T10 are designed to demonstrate exactly this.
- **Improves portability:** skills/AGENTS.md/mcpServers adoption; ACP roles.
- **Improves long-run reliability:** per-step persistence (universal), retry taxonomies, budgets, event-sourced state (OpenHands), recipe retry gates (goose). These become *essential* only for unattended/multi-user/remote operation — exactly the regime mini-swe never enters (its batch runner externalizes reliability to task-level parallelism).
- **Unjustified abstraction (candidates):** double MCP clients (cline), three concurrent compaction mechanisms (goose), experimental server layers shipped alongside stable ones (pi CBOR, cline legacy hostbridge), 150-provider catalogs on 9 code paths (cline). Each is CONFIRMED present; whether it *costs* outcomes is unmeasurable statically.
- **In-repo eval methodology critique:** OpenHands' integration/behavior tests use an LLM judge without budget controls; goose's evals (harbor) and cline's evals run against their own CLIs without cross-harness controls; only mini-swe ships a leaderboard-grade harness. No subject's published numbers are cross-comparable (different models/budgets/retries) — repeated per evidence rule §1.8 wherever cited.

**Frontier map:** *Commoditized:* native tool-calling loops, provider gateways, streaming, per-step persistence, ripgrep search, exact-match editing, retries, token accounting. *Emerging standards (real, versions skewing):* MCP (2024-11-05…2025-11-25 negotiated), ACP (v1, churn documented), SKILL.md, AGENTS.md, mcpServers JSON. *Project-specific:* recipes+scheduler (goose), event-tree state + engine-in-container (OpenHands), permission-pattern inference via tree-sitter (opencode), injectable loop seams (pi), canonical-transcript compaction + cron specs (cline). *Labels that outrun the code:* "minimal" (both minimal subjects are large projects), "sandboxed" when it means prompts, provider counts. *Unsolved:* default-on local isolation; mechanical requirement-verification; blocking injection defense; compaction that provably preserves task-critical state; cross-harness benchmark comparability.

**Coherence verdict on the emerging stack** (AGENTS.md + Skills + MCP + ACP + LSP/indexes + containers + file state + tests): **no single project implements it coherently.** Closest is OpenHands (7 of 8 layers native or partial — missing LSP; containers native) followed by goose (missing LSP, containers opt-in) and opencode (only LSP-native subject; missing containers, ACP client role). The stack exists today only as fragments that happen to compose at the file layer — which is precisely what makes the §5 boundary viable.

## 5. Portability-boundary verdict (prompt §5)

**The proposed boundary is correct, with two amendments.** Confirmed: Skills hold procedure (5/6 native, format-portable), AGENTS.md holds policy (5/6), plain files hold state (universal; also the only compaction-proof context), git holds history (universal; three subjects add their own git layers without conflicting — opencode side-repo snapshots, cline private refs, OpenHands git module), MCP optional (4/6), tests as external truth (universal executor: shell). Amendments: **(1)** the harness must not own *verification semantics* either — but today gates can only be enforced with native mechanisms (Stop hooks, retry-checks, ToolPolicies), so the "thin adapter" must include the gate wiring, not just prompt plumbing; **(2)** "harness owns execution" must be read as "harness owns *host* execution" — isolation is a separate, user-supplied layer everywhere except OpenHands.

Per-harness adapter table (mechanism → what it silently fights → unattended coverage; citations in dossiers §H):

| Harness | Thin-adapter mechanism (native) | Silently fights | Unattended coverage |
|---|---|---|---|
| opencode | `.opencode/skill/`, AGENTS.md, permission ruleset in opencode.json, plugin `permission.ask`/`tool.execute.*` hooks, markdown agents (plan agent edit-denied) | auto-compaction pruning tool evidence + auto-continue drift (disable via config); todowrite shadowing file todos (deny it); doom-loop asks under non-auto | high |
| goose | recipe YAML + sub_recipes + `retry.checks` shell gates; `.agents/skills`, AGENTS.md/.goosehints; Stop-hook as verification veto; `goose review` checks | 3 compaction passes rewriting visible history; hidden goal/grind user-role nudges; moim token overhead (partly non-disableable); SmartApprove LLM judge stalls | high |
| OpenHands | SKILL.md + AGENTS.md-as-skill; subagent md frontmatter; shell hooks (Stop veto); `NeverConfirm`; DockerWorkspace for containment | condenser forgetting in-context task state @240 events (keep state in files); stuck detector nudging legitimate test-fix loops; memory loader injecting MEMORY.md | high (highest with isolation) |
| pi | AGENTS.md; shared skills dirs; extension for gates (`tool_call` block); `--mode json`/RPC as outer driver; `--no-*` isolation flags | essentially nothing (no competing machinery) — but supplies no budgets/gates either; skills need forced loading (`/skill:name`) | high, DIY enforcement |
| mini-swe | YAML config + instance/observation templates (the whole surface) | nothing structural; also *helps* with nothing — model self-polices every step; sentinel submit can fire with steps skipped | possible but crude |
| cline | AGENTS.md + `.cline/{rules,workflows,skills}`; subprocess hooks; custom ToolPolicies via SDK (CLI yolo swaps the system prompt — avoid for "unattended but careful"); cron md specs | agentic compaction summarizing tool results; plan/act envelope tags; checkpoint refs polluting `refs/cline/*`; SDK auto-approve default | high |

## 6. Recommendations by audience

1. **Individual developer:** opencode (terminal-first) or cline (IDE-first) — both give strong daily ergonomics; add your own container if you run untrusted code. pi if you enjoy building your harness.
2. **Team standardizing workflow:** goose (recipes + review checks + scheduler + keyring) or opencode (server + permission rulesets shipped to every seat). Encode the workflow in skills/AGENTS.md first so the harness choice stays reversible (§5).
3. **Org building an internal platform:** OpenHands SDK — the agent-server/workspace split is the only shipped control plane with real isolation; expect to build multi-user auth (flat API keys today) and pin the litellm/acp dependency skew.
4. **Startup building a hosted autonomous developer:** OpenHands SDK for the runtime + cline's architecture as the reference for engine-as-product packaging; do not build your own loop — build verification and isolation, the two unsolved layers (§4).
5. **Harness researcher:** mini-swe-agent as control (record: tool-calling default, 190 lines), pi for loop experiments without forking; benchmark plan and fairness caveats in 10.
6. **Portable-Skills author:** write for lazy, model-initiated loading (the weakest activation model — pi's); keep procedures re-entrant and state in files; test on OpenHands (trigger-injection) and opencode (skill tool) as the two extremes; never depend on native todos/plan modes.
7. **Composer of a new system:** the working composition today is goose (broker/orchestration) → opencode or Claude-family agents via ACP (coding loops) → inside OpenHands DockerWorkspaces (isolation) → skills/AGENTS.md/files/tests (portable methodology) → mini-swe (measurement). Every arrow verified in 09; the goose→OpenHands arrow is the one that needs an adapter.

## 7. Completion-gate gap analysis

Deliverables: 00 (dashboard, live) · 01 (draft-complete) · 02 (draft-complete, 10 groups × 6) · 03–08 (draft-complete, all spot-checked ≥3, 26/26 passed) · 09 (draft-complete, 11 combos + diagram) · 10 (draft-complete, all six invocation sections) · 11 (350 merged rows + 3 external additions) · 12 (this file). Prompt passes: A/B per-repo ✔ (dossiers §A/§B) · C ✔ (02) · D ✔ (dossiers §D) · E ✔ (dossiers §E, all findings tagged CONFIRMED/RISK) · F ✔ (09) · G ✔ (§4 above + 10). Twenty questions: all answered above; #11/#19/#20 answered as hypotheses with named experiments (allowed by prompt §4). §5 methodology assessment ✔ (§5 above + dossiers §H).

**Unresolved items and what would resolve them:**
1. Outcome claims (which features improve success/cost) — resolved only by executing 10-benchmark-design (controlled model, metering proxy, ≥3 seeds).
2. OpenHands-CLI repo not in pinned scope — clone+pin it to confirm "CLI = SDK consumer" (currently doc-level, marked UNVERIFIED in 05).
3. cline: whether the browser tool reaches SDK sessions; prompt-caching insertion end-to-end — targeted code trace (08 §follow-up work).
4. goose: `agents/snapshots` checkpoint semantics; exact block→aaif rename date — one code read + `gh` history check (04 §follow-up work).
5. opencode: `packages/llm` native runtime depth; full httpapi endpoint list — follow-ups listed in 03 §follow-up work.
6. Live interop pairings (goose→opencode ACP; opencode-in-OpenHands-sandbox; ACPAgent+opencode) — 30-minute smoke tests each, specified in 09; would upgrade three "works today" claims from handshake-verified to run-verified.
7. MCP 2026-07-28 adoption — re-check dependency bumps at a future pin; all six lag the spec at this one (external, ledger).
