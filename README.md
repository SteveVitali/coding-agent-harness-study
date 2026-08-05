# Coding-Agent Harness Study

A pinned-commit comparative field study of six open-source AI coding-agent harnesses — **opencode**, **goose**, **OpenHands (app + SDK)**, **pi-mono**, **mini-swe-agent**, and **cline** — conducted by reading the code, not the READMEs.

Every load-bearing claim in this repository is cited as `repo-relative/path:lines` against a specific pinned commit, recorded in a [350-row evidence ledger](research/11-evidence-ledger.md), and subject to the verification protocol described in [METHODOLOGY.md](METHODOLOGY.md) (36/36 citation spot-checks passed). Snapshot date: **2026-08-04**.

> **Staleness warning.** These are fast-moving projects (some merge 400+ commits/month). Every finding is true *at the pinned commit* and may already be stale upstream. Corrections and drift reports are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

## Headline findings

**Six projects, four architectural species** ([synthesis §1](research/12-final-synthesis.md)):

- **opencode** — a *product platform*: one heavily-engineered TS engine behind an HTTP API, seven first-class UIs, and more model-quirk handling than anyone.
- **goose** — a *protocol broker*: the only subject implementing MCP in both roles **and** ACP in both roles; it can run competitors' agents as its "model."
- **OpenHands (SDK + Agent Canvas)** — an *engine/runtime/console stack*: the only subject where the sandbox *is* the deployment unit (whole engine in a container behind REST/WS).
- **pi-mono** — a *construction kit*: the cleanest injectable loop, with an authored refusal to own MCP, permissions, subagents, plan mode, or todos.
- **mini-swe-agent** — the *control arm*: a 190-line agent class, one bash tool, and the study's only leaderboard-grade benchmark harness.
- **cline** — an *engine-as-product inversion*: the VS Code extension is now a host adapter over published npm packages.

**Two gaps shared by all six** ([synthesis Q17](research/12-final-synthesis.md)):

1. **No OS-enforced isolation on the default local path.** Every subject defaults to host execution with prompt-level authority; OS enforcement exists only where a container is explicitly chosen. Permission prompts, allow/deny rules, and LLM safety judges are all in-process mechanisms that an approved shell command can route around — several projects say so themselves in code comments or docs.
2. **Nobody mechanically distinguishes "command exited 0" from "requirement demonstrated."** All six equate exit status or model assertion with success; verification gates exist only as harness-specific hooks the user must wire up.

**Four stale-classification corrections** — cases where the study's own starting assumptions (and often the projects' docs) were wrong at the pin ([details](research/00-scope.md)):

- **OpenHands/OpenHands is no longer the engine.** At the pin it is *Agent Canvas*, a TS/React/Electron frontend with no Python engine; the SDK repo is the sole authoritative engine, and the sandbox/runtime lives there.
- **mini-swe-agent's "no tool-calling API" is stale.** The v2 default uses native tool-calling with a single `bash` tool; the text-regex path survives as opt-in. "~100 lines" is 190 for the agent class.
- **cline is SDK-first, not VS Code-first.** Published `@cline/*` npm packages with a zero-VS-Code-import CLI; the extension is a host adapter. The legacy XML parsing loop is gone.
- **goose's governance moved further than reported:** the pinned tree identifies as `aaif-goose/goose`, authored by the Agentic AI Foundation at the Linux Foundation; its bespoke REST server is gone — ACP *is* the API.

**Portability-boundary verdict** ([synthesis §5](research/12-final-synthesis.md)): a harness-independent methodology layer — skills for procedure, AGENTS.md for policy, plain files for state, git for history, tests as external truth — is viable today across 5/6 subjects, with two amendments: the thin adapter must include verification-gate wiring (gates are only enforceable with harness-native mechanisms), and "harness owns execution" means *host* execution — isolation is a separate, user-supplied layer everywhere except OpenHands.

## The twenty questions

One-line verdicts; confidence ratings, supporting sections, and the full table live in [research/12-final-synthesis.md](research/12-final-synthesis.md).

| # | Question | Verdict |
|---|---|---|
| 1 | Best daily coding agent | opencode — deepest interactive surface with serious loop tests |
| 2 | Strongest general-purpose shell | goose — MCP×2 + ACP×2 + recipes + scheduler; "agent of agents" is shipped |
| 3 | Best production runtime for hosted coding agents | OpenHands SDK + agent-server — container-as-engine, REST/WS control plane |
| 4 | Best foundation for a custom harness | pi-agent-core (loop control); cline SDK (batteries + npm); OpenHands SDK (Python/infra) |
| 5 | Best experimental baseline | mini-swe-agent — by design and by output format |
| 6 | Strongest context engineering | OpenHands — condenser + cached system prompt + triggered skills + memory |
| 7 | Strongest security/sandboxing | OpenHands (hosted) / mini-swe environments (batch); no subject sandboxes the default local path |
| 8 | Strongest extension model | goose — everything is a protocol |
| 9 | Cleanest internal architecture | pi; cline SDK second; the two biggest loops are monoliths |
| 10 | Most dangerous hidden assumptions | model-self-assessed safety (OpenHands default risk, goose LLM judge, cline auto-approve defaults, opencode model-ID forks) |
| 11 | Most harness overhead | goose — up to 4 hidden LLM side-calls per turn (hypothesis → benchmark) |
| 12 | Genuinely portable components | SKILL.md, AGENTS.md, mcpServers JSON, files+git state, mini-swe trajectory format |
| 13 | Technically coherent combinations | goose→opencode via ACP; opencode inside OpenHands sandbox; Canvas over any ACP agent |
| 14 | Redundant/overengineered combos | goose⇄OpenHands orchestration today; any double-harness stack where one engine + a container suffices |
| 15 | Unified best-of-all-worlds feasible? | as one codebase: no; as a composition over the file/protocol layer: yes |
| 16 | What would it look like | stateless injectable loop + OpenHands-style isolation + goose-style brokerage + portable file-layer methodology |
| 17 | Missing from all six | OS-enforced default isolation; mechanical requirement-verification; blocking injection defense |
| 18 | Most likely to stay strategically relevant | OpenHands SDK; goose's broker position second |
| 19 | Benefits most from strong models | mini-swe-agent and pi (hypothesis → benchmark) |
| 20 | Adds most value beyond the model | opencode (hypothesis → benchmark) |

## Repository map

Read in numeric order for the full arc, or jump straight to the synthesis.

| File | What it is |
|---|---|
| [research/00-scope.md](research/00-scope.md) | Study dashboard: pinned commits, verification checklist, classification corrections, open questions |
| [research/01-repository-inventory.md](research/01-repository-inventory.md) | Cross-repo vitals, classification verdicts, one-paragraph architectural maps |
| [research/02-comparison-matrix.md](research/02-comparison-matrix.md) | 10 dimension groups × 6 subjects (product, model layer, tools, context, planning, execution, security, verification, extensibility, DX) |
| [research/03-opencode.md](research/03-opencode.md) | Dossier: sst/opencode (anomalyco) |
| [research/04-goose.md](research/04-goose.md) | Dossier: aaif-goose/goose (formerly block/goose) |
| [research/05-openhands.md](research/05-openhands.md) | Dossier: OpenHands app (Agent Canvas) + software-agent-sdk, analyzed as one subject |
| [research/06-pi-mono.md](research/06-pi-mono.md) | Dossier: badlogic/pi-mono |
| [research/07-mini-swe-agent.md](research/07-mini-swe-agent.md) | Dossier: SWE-agent/mini-swe-agent |
| [research/08-cline.md](research/08-cline.md) | Dossier: cline/cline |
| [research/09-interoperability.md](research/09-interoperability.md) | 11 cross-project pairings assessed + the layer diagram (renders on GitHub) |
| [research/10-benchmark-design.md](research/10-benchmark-design.md) | **A runnable benchmark design, not results** — task suite, controls, metering, per-harness invocation, fairness hazards |
| [research/11-evidence-ledger.md](research/11-evidence-ledger.md) | The 350-row claim-by-claim evidence ledger (raw per-repo fragments in [ledger-fragments/](research/ledger-fragments/)) |
| [research/12-final-synthesis.md](research/12-final-synthesis.md) | Executive synthesis, twenty questions, tradeoffs, frontier map, portability verdict, recommendations by audience |
| [PROMPT.md](PROMPT.md) | The verbatim research prompt that drove the study — its protocol registration |
| [METHODOLOGY.md](METHODOLOGY.md) | Evidence rules, verification protocol, AI-conduct disclosure, reproduction instructions |

Each dossier follows the same structure: §A orientation → §B one agent turn traced keystroke-to-persisted-state → §C matrix inputs → §D governing philosophy → §E hidden tradeoffs (tagged CONFIRMED/RISK) → §F interop surfaces → §G/H fit assessments.

## Reproducing and verifying claims

Citations use `repo-relative/path.ext:line-range` pinned to these commits. Local paths like `~/opencode` in the research files refer to the investigator's clones of:

| Subject | Upstream | Pinned commit |
|---|---|---|
| opencode | [sst/opencode](https://github.com/sst/opencode) (repo metadata at pin: anomalyco/opencode) | `66fdd51f0d6db8e47e876721c855ea155043b74c` |
| goose | [block/goose](https://github.com/block/goose) (repo metadata at pin: aaif-goose/goose) | `7e431ac6f804fdc5a6fb9262fa2ca5b8b0fd6ce6` |
| OpenHands app | [OpenHands/OpenHands](https://github.com/OpenHands/OpenHands) | `2c9ba42b19c7d3dc115f15b49acd8c735dcad674` |
| OpenHands SDK | [OpenHands/software-agent-sdk](https://github.com/OpenHands/software-agent-sdk) | `0c8f97aab8a22d438bdea45ae3963e6050a9374c` |
| pi-mono | [badlogic/pi-mono](https://github.com/badlogic/pi-mono) | `588915ec71714688cee8b7153339e8bdebb3e82e` |
| mini-swe-agent | [SWE-agent/mini-swe-agent](https://github.com/SWE-agent/mini-swe-agent) | `a83fcae82d2a08f0ee0c688f9d137b3566c097f8` |
| cline | [cline/cline](https://github.com/cline/cline) | `64993e78d5623920e1fb549fb20b6d15ed9b710a` |

To check any citation:

```sh
git clone https://github.com/sst/opencode && cd opencode
git checkout 66fdd51f0d6db8e47e876721c855ea155043b74c
sed -n '30,37p' packages/opencode/src/tool/websearch.ts   # example: ledger row on websearch routing
```

## How this study was conducted

This investigation was **conducted by AI coding agents** (Claude-family models running in an agentic coding harness), human-directed, over multi-hour sessions on 2026-08-04. One subagent per repository produced the dossiers; the coordinating session performed cross-repo synthesis and verified subagent citations against the actual files. The full protocol — evidence rules, the files-as-source-of-truth working discipline, the citation spot-check and completion-gate procedure — is in [METHODOLOGY.md](METHODOLOGY.md), and the prompt that specified it is preserved verbatim in [PROMPT.md](PROMPT.md).

The method is part of the point: this repo doubles as a worked example of evidence-ledger-driven multi-agent research with mechanical verification of the agent's own citations.

## A note on tone

Several findings concern security posture, hidden LLM side-calls, or documentation drift. They critique *architectures at a pinned commit*, not maintainers. All six projects held up impressively under close read — each dossier's §E includes positive CONFIRMED findings, and the tradeoff analysis treats every design cost as the price of a deliberate bet. Where a project documents its own limitation honestly, the study says so.

Quoted code snippets throughout are ≤3 lines each, attributed by path and line to their upstream repositories, and remain under their upstream licenses (MIT or Apache-2.0 across all six subjects).

## Contributing

Found a citation error, or evidence that a finding has drifted upstream? Open an issue with a `path:line` citation — same rules the study held itself to. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

[MIT](LICENSE) for the text of this study. Upstream code snippets remain under their projects' licenses, attributed in place.
