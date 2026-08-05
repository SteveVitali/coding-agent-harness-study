<!--
This is the verbatim research prompt (v2, 2026-08-04) that drove the study in
this repository — its protocol registration. It was pasted as-is into fresh
agent sessions; the deliverables it specifies are the files under research/.
Preserved unedited except for this preamble. How its execution was verified is
documented in METHODOLOGY.md.
-->

# Deep Comparative Investigation: Open-Source Coding-Agent Harnesses

You are conducting a multi-session comparative investigation of six open-source AI coding-agent projects. The end product must be rigorous enough to drive an architectural decision about which project (or combination) to build on, and how to layer a portable, harness-independent implementation methodology on top.

## 0. Subjects, pinned revisions, and ground truth

All repositories are already cloned under `~/` and pinned to the commits below (cloned 2026-08-04; confirm with `git -C <repo> rev-parse HEAD` and record in `research/00-scope.md`). Every file/line citation you make is implicitly pinned to that SHA. Do not `git pull` during the investigation.

| Local path | Upstream | Pinned commit | Provisional classification (to verify) |
|---|---|---|---|
| `~/opencode` | sst/opencode (Anomaly) | `66fdd51f0d6d` | Productized terminal coding agent; TypeScript/Bun rewrite; client/server split; embeddable server. |
| `~/goose` | block/goose | `7e431ac6f804` | Rust + MCP/ACP-composition general agent; now a Linux Foundation project. |
| `~/OpenHands` | OpenHands/OpenHands | `2c9ba42b19c7` | Hosted/autonomous agent app: runtime, sandboxes, deployment. |
| `~/openhands-software-agent-sdk` | OpenHands/software-agent-sdk | `0c8f97aab8a2` | The V1 engine behind OpenHands CLI/Cloud — analyze together with `~/OpenHands` as one subject; determine what lives where and which is authoritative. |
| `~/pi-mono` | badlogic/pi-mono | `588915ec7171` | Minimal, extensible TypeScript harness construction kit (pi coding agent + agent runtime + TUI libs). |
| `~/mini-swe-agent` | SWE-agent/mini-swe-agent | `a83fcae82d2a` | ~100-line research baseline from the SWE-bench team; bash-only, no tool-calling API. |
| `~/cline` | cline/cline | `64993e78d562` | VS Code-first coding product; verify the claim that its engine is separately consumable (SDK/core package) rather than assuming it. |

Corrections to stale priors — verify against the code, but do not "rediscover" the old state: OpenCode is no longer the Go/Charm codebase (that fork became Crush); OpenHands V1 split its engine into the SDK repo; Goose governance moved to the Linux Foundation. Treat all classifications above as hypotheses to confirm, refine, or reject with cited evidence.

**Explicitly out of scope:** the leaked Claude Code source (March 2026 npm source-map leak) and any mirror or derivative of it. It is DMCA'd proprietary code and known mirrors carry malware; do not clone, fetch, or cite it. Where a Claude Code comparison is useful, cite only Anthropic's public documentation, and label those points "documentation-level evidence." Aider, Codex CLI, Qwen Code, Kilo Code, Continue, and Crush are acknowledged peers but out of scope; mention them only when a subject explicitly borrows from or interoperates with them.

## 1. Evidence rules (these override everything else)

1. Cite as `repo-relative/path.ext:line-range` and only for lines you have actually read in this investigation. Never cite from memory of a project.
2. Every load-bearing claim gets a row in `research/11-evidence-ledger.md` **at the moment you form it**, with: claim · repo · path:lines · ≤3-line verbatim snippet · evidence class (`impl` / `test` / `config` / `doc` / `commit-history` / `inference`) · confidence (`high` / `med` / `low`).
3. Implementation beats documentation; tests beat both when they conflict. Record contradictions in the ledger instead of smoothing them over.
4. Anything you could not verify in code gets an explicit `UNVERIFIED:` prefix wherever it appears. An unverified claim may not appear in a recommendation.
5. GitHub issues/discussions are **not** in the clones. You may use `gh` read-only (`gh issue list`, `gh api`, `gh release view`) and targeted web fetches for: issue/PR discussions, release notes, and protocol specs (MCP, ACP, AGENTS.md, Agent Skills). Label all such evidence `external` and date it. Never let external material override what the pinned code says.
6. Commit history evidence comes from `git log`/`git blame` on the pinned clones.
7. Do not equate stars with quality, feature count with capability, minimalism with efficiency, protocol mentions with interoperability, permission prompts with sandboxing, or a passing command with a proven requirement.
8. Benchmark or quality claims (yours or the projects') derived from different models/budgets/setups are not comparable; say so wherever you repeat one.

## 2. Working protocol (context durability and effort budget)

This investigation will outlive any single context window. Therefore:

- **Files are the source of truth, not your context.** After each work increment, write findings into the appropriate `research/` file before moving on. Assume your in-context memory of a repo will be lost; anything not written down is wasted work.
- Every research file starts with a status header: `Status: not-started | in-progress | draft-complete | verified`, `Last updated`, and `Next steps` — so a resumed session (or a fresh one) can continue without re-reading the repos from scratch.
- `research/00-scope.md` is the dashboard: pinned SHAs, pass-by-pass progress checklist, open questions, and deviations from this prompt. Update it whenever you finish or reprioritize a pass.
- **Delegate mechanically-heavy exploration to subagents** (one per repo for Pass A/B fact-finding), each instructed to return path-cited findings in the dossier structure below. Reserve your own context for cross-repo synthesis, verification of subagent citations (spot-check at least 3 citations per dossier against the actual files), and the final deliverables.
- **Depth budget:** OpenHands(+SDK), OpenCode, and Goose are the largest and warrant the deepest tracing. mini-SWE-agent should take an hour, not a day — read all of it. Sample, don't enumerate: trace **one representative agent turn** end-to-end per project rather than every code path; inspect the 2–3 most instructive tools (shell exec, file edit, and one extension-provided tool), not the whole catalog.
- If a section of this prompt proves inapplicable to a project (e.g., "Kubernetes execution" for mini-SWE-agent), write one line saying so and move on. Absence findings are findings.

**Pre-existing scaffold:** `research/00-scope.md` (dashboard with pinned SHAs and dated protocol reference points) and `research/10-benchmark-design.md` (task suite, controls, metrics; per-harness invocation sections still empty) already exist from an earlier session. Continue and update them in place — do not recreate or duplicate them.

## 3. Research passes

Passes A–B fill the per-repo dossiers (`research/03`–`08`). Passes C–G are cross-repo and fill the remaining files. Interleave freely, but keep the dashboard honest.

### Pass A — Orientation (per repo)

Identify: languages/frameworks; package layout; entry points; public APIs; user-facing surfaces; runtime components; extension mechanisms; deployment modes; test organization; docs structure; release/versioning model; license; maturity signals (commit cadence, CI, changelog discipline); and degree of coupling to a commercial product or hosted service. Produce a one-page architectural map per repo. Do not rely on filenames — open the files.

### Pass B — Reconstruct one agent turn (per repo)

Trace a single representative turn from keystroke to persisted state, citing the concrete symbol at each stage: input ingestion → system/project-instruction assembly → context selection → request construction → provider abstraction → streaming → tool-call parsing → authorization → tool execution locus → observation return → loop termination → state persistence → compaction/pruning → failure/retry/cancellation/timeout handling. "The framework manages tools" is a forbidden sentence; name the component.

### Pass C — Comparative matrix

Fill `research/02-comparison-matrix.md` across these dimension groups (each cell: `native` / `via-extension` / `partial` / `external` / `absent` / `unclear`, plus a note wherever a label alone would mislead):

- **Product & interface:** TUI; IDE integration; desktop/web UI; headless/CI; server API; SDK quality; multi-session; session branching/forking; approval UX; diff/checkpoint UX.
- **Model layer:** provider abstraction and where it leaks; provider classes incl. OpenAI-compatible and local; model-specific adaptations; reasoning-model support; prompt caching; token/cost accounting; retry/fallback; whether the harness can drive another complete agent (not just another model).
- **Tool system:** built-ins; registration and schemas; MCP (client? server? which spec revision?); ACP (which role?); LSP; browser/web tools; shell execution; file-edit mechanism (patch format, fuzzy matching, conflict handling); git integration; custom-tool API; plugin lifecycle; tool-result truncation/normalization.
- **Context engineering:** repo maps; search/retrieval; explicit file selection; LSP-derived context; Agent Skills or equivalent; AGENTS.md or equivalent; memory; session history; compaction/summarization (what is preserved vs lost); context isolation; durable external state; large-repo and long-task behavior.
- **Planning & agency:** plan vs build modes; todos; subagents (real processes/contexts, or prompt templates?); delegation; parallelism; workflow graphs; recipes/declarative tasks; background execution; scheduling; stopping conditions; recovery from interrupted runs.
- **Execution environment:** host exec; containers; sandboxes (which technology — seatbelt/landlock/docker/gVisor?); remote workers; Kubernetes; filesystem/network isolation; credential handling; reproducibility; multi-user architecture.
- **Security & authority:** permission model; allow/deny/ask rules and command-pattern matching; path and network restrictions; prompt-injection defenses; auditability; reversibility; default trust posture; **and the key question: is each boundary enforced by the OS/runtime, or is it a UI prompt the model output could route around?**
- **Verification:** test/lint/typecheck execution; validation loops; independent review; acceptance-criteria tracking; evidence capture; benchmark integration; whether the system distinguishes "command exited 0" from "requirement demonstrated."
- **Extensibility & portability:** plugin APIs and their stability; skills; MCP servers; ACP endpoints; hooks/events; custom UIs/agents/tools/providers/runtimes; config portability; lock-in severity.
- **Developer experience:** setup cost; config clarity; code readability; abstraction quality; testability; debuggability; effort to add a tool / add a provider / change the loop / embed / ship to a team.

### Pass D — Governing philosophy (per repo)

Infer the design philosophy the code embodies (batteries-included, minimal-mechanism, protocol composition, secure remote execution, editor-first, benchmark simplicity, …) and support it with architectural evidence. State which complexities each project owns and which it deliberately pushes onto users, plugins, or infrastructure.

### Pass E — Hidden tradeoffs (per repo)

Hunt for costs invisible in marketing: duplicated client/server state; UI–runtime coupling; leaky provider abstractions; weak cancellation; covert dependence on one model family; permissive tool execution; prompt-only "permissions"; brittle parsing; lossy compaction; unstable plugin surfaces; prompt-template "subagents"; harness-locked orchestration formats; token overhead; poor observability; thin tests around the core loop. **Tag each finding `CONFIRMED` (with citation) or `RISK` (plausible, mechanism stated, no direct evidence).** Do not report a defect without one of those tags.

### Pass F — Interoperability

For each candidate combination (Goose↔OpenCode via ACP; Goose orchestrating OpenHands workers; OpenCode inside an OpenHands sandbox; Pi prototype → OpenHands deployment; Cline engine embedded elsewhere sharing MCP servers; Skills reused across harnesses; AGENTS.md as common policy; MCP as shared tool layer; ACP as agent/client layer; mini-SWE-agent as measurement baseline; a portable workflow layered over all six): determine whether it works **today**, via which implemented protocol roles/versions/capabilities (inspect the handshake code, not the README badge); where adapters are needed; what semantic mismatches remain (state, permissions, context, lifecycle); and whether it is *useful* or merely possible. → `research/09-interoperability.md`.

### Pass G — Complexity vs. demonstrated benefit, and the frontier

Using mini-SWE-agent as the null hypothesis, classify each major architectural feature as improving: task success, usability, safety, portability, long-run reliability — or as product surface / unjustified abstraction — and note which capabilities become essential only for long-running, unattended, multi-user, or remote operation. Where repos contain evals, critique their methodology (controls for model, budgets, tools, environment, retries, task selection). **Static code reading cannot prove outcome claims: phrase outcome conclusions as hypotheses and route each one to a concrete experiment in `research/10-benchmark-design.md`.**

Then describe the frontier: which capabilities are commoditized, emerging standards, project-specific, mostly marketing, or unsolved. Evaluate specifically whether the emerging stack — AGENTS.md (repo policy) + Agent Skills (procedures) + MCP (tools) + ACP (agent/client interop) + LSP/indexes (code intelligence) + containers (isolation) + filesystem artifacts (durable state) + tests (external truth) — is implemented *coherently* by any one project, or exists only as disjoint fragments.

## 4. The twenty questions

The final synthesis must answer each with: **a one-line verdict + confidence (high/med/low) + pointer to supporting dossier/ledger entries.** Render as a table first, prose after.

1. Best open-source daily coding agent. 2. Strongest general-purpose agent shell. 3. Best production runtime for hosted coding agents. 4. Best foundation for a custom harness. 5. Best experimental baseline. 6. Strongest context engineering. 7. Strongest security/sandboxing. 8. Strongest extension model. 9. Cleanest internal architecture. 10. Most dangerous hidden assumptions. 11. Most harness overhead. 12. Genuinely portable components. 13. Technically coherent combinations. 14. Redundant/overengineered combinations. 15. Is a unified best-of-all-worlds architecture feasible? 16. What would it look like? 17. What major capability is missing from all six? 18. Most likely to stay strategically relevant as models improve. 19. Which benefits most from strong models? 20. Which adds the most value beyond the model?

Where a question is genuinely undecidable from static analysis, say so and name the experiment that would decide it — do not manufacture a confident answer.

## 5. Portable methodology assessment

I am building a harness-independent procedure taking a spec through: requirements reconstruction → implementation planning → test derivation → implementation → self-review → gap analysis → unit/integration testing → live verification → reviewable PR with evidence mapped to acceptance criteria.

Candidate portability boundary: Skills hold the canonical procedure; AGENTS.md holds repo policy; plain files hold durable task state; git holds reversible history; MCP provides optional capabilities; harness-native commands/recipes/plugins act **only as thin adapters**; tests provide external truth; the harness owns execution but never the methodology.

Assess whether that boundary is correct, and per project: exactly which native mechanism the adapter would use (name the config file / plugin API / recipe format, with citations), what the harness would silently fight you on (auto-compaction discarding task state, native todo systems shadowing your files, permission models blocking the verification steps), and how much of the methodology each harness could execute unattended.

## 6. Deliverables

Maintain these continuously (statuses in each header):

- `research/00-scope.md` — dashboard: SHAs, progress, deviations.
- `research/01-repository-inventory.md` — Pass A maps.
- `research/02-comparison-matrix.md` — Pass C.
- `research/03-opencode.md` … `08-cline.md` — dossiers, one per subject (OpenHands app+SDK share `05`): architectural overview; agent-turn walkthrough; key modules; major abstractions; extension points; execution model; state model; context strategy; permission model; strengths; weaknesses; hidden tradeoffs (tagged); best-fit and poor-fit use cases. Target ≤400 lines each — dense and cited, not exhaustive.
- `research/09-interoperability.md` — Pass F + a Mermaid diagram layering: model / agent-loop / workflow / tool / client / runtime / verification, showing which project occupies which layer and which protocols bridge them.
- `research/10-benchmark-design.md` — a runnable local benchmark plan: same tasks through multiple harnesses (short bug fix; repo exploration; multi-file feature; spec-driven implementation; test repair; refactor; long task requiring durable state; task embedding adversarial repo instructions; task requiring MCP tools; task that should be refused/escalated). Control model, repo state, and budgets. Measure: completion, wall-clock, tokens, cost, tool calls, unnecessary actions, files changed, test outcomes, requirement coverage, interventions, compaction events, retries, verification quality, permission violations, diff quality. Specify per-harness invocation commands and where each harness makes fair comparison impossible.
- `research/11-evidence-ledger.md` — per §1, appended continuously.
- `research/12-final-synthesis.md` — executive synthesis (which layer each project occupies and how they differ, ≤2 pages); the twenty-question table; tradeoff analysis (minimality↔safety, ergonomics↔embeddability, generic protocols↔deep integration, model freedom↔model-specific tuning, autonomy↔approval, remote isolation↔local responsiveness, declarative↔programmable, subagent specialization↔token overhead); portability-boundary verdict (§5); and recommendations for seven audiences: individual developer; team standardizing workflow; org building an internal platform; startup building a hosted autonomous developer; harness researcher; portable-Skills author; and someone composing a new system from the best components.

## 7. Completion gate

Before declaring the investigation done: reread this prompt top to bottom and write an explicit gap analysis at the end of `12-final-synthesis.md` — every numbered question and deliverable, where it was answered, and for anything unresolved, what experiment or information would resolve it. Then spot-check ten random ledger citations against the pinned clones; if more than one fails, audit the whole ledger. The bar: a reader should be able to make the build-vs-adopt decision from your files alone, without trusting your memory.
