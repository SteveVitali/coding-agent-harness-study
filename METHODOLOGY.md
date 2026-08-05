# Methodology

This document describes how the study was conducted, the rules the evidence had to satisfy, how the findings were verified, and how to reproduce or challenge them. The prompt that specified this protocol is preserved verbatim in [PROMPT.md](PROMPT.md); this file summarizes and reports on its execution.

## 1. Study design

- **Subjects:** six open-source coding-agent harnesses (seven repositories — OpenHands' app and SDK repos are analyzed as one subject), each cloned and pinned to a specific commit on 2026-08-04. Pinned SHAs are in the [README](README.md) and [research/00-scope.md](research/00-scope.md). No `git pull` during the investigation.
- **Method:** static close-reading of the pinned code, structured as research passes: per-repo orientation (A) and single-agent-turn trace (B); cross-repo comparison matrix (C); governing-philosophy analysis (D); hidden-tradeoff hunting (E); interoperability assessment (F); complexity-vs-benefit analysis with a benchmark design for claims that static reading cannot decide (G).
- **Out of scope:** executing the benchmark (the design is in [research/10-benchmark-design.md](research/10-benchmark-design.md)); the leaked Claude Code source (DMCA'd, explicitly excluded by the protocol); peer projects (Aider, Codex CLI, Crush, etc.) except where a subject explicitly interoperates with them.

## 2. Who conducted it

AI coding agents (Claude-family models in an agentic coding harness), directed by a human investigator. The division of labor:

- **One subagent per repository** performed the mechanically-heavy exploration (Passes A/B/E fact-finding), instructed to return path-cited findings in a fixed dossier structure and to write an evidence-ledger fragment as claims were formed.
- **The coordinating session** performed cross-repo synthesis (files 01, 02, 09, 10, 12), merged the ledger, and — critically — **verified subagent citations against the actual files** rather than trusting them.
- **Files, not context, were the source of truth.** Every increment of work was written to `research/` before moving on, on the assumption that any session's in-context memory would be lost. The working dashboard ([research/00-scope.md](research/00-scope.md)) is preserved as part of the record.

## 3. Evidence rules

The rules, as specified in the protocol (PROMPT.md §1) and applied throughout:

1. Cite as `repo-relative/path.ext:line-range`, only for lines actually read during the investigation. Never cite from memory of a project.
2. Every load-bearing claim gets a row in the [evidence ledger](research/11-evidence-ledger.md) *at the moment it is formed*, with: claim · repo · path:lines · ≤3-line verbatim snippet · evidence class (`impl` / `test` / `config` / `doc` / `commit-history` / `inference` / `external`) · confidence (`high` / `med` / `low`).
3. **Implementation beats documentation; tests beat both** when they conflict. Contradictions are recorded in the ledger, not smoothed over.
4. Anything not verified in code carries an explicit `UNVERIFIED:` prefix and may not appear in a recommendation.
5. External evidence (GitHub issues, release notes, protocol specs, npm registry checks) is labeled `external` and dated; it never overrides what the pinned code says.
6. Forbidden equivalences: stars ≠ quality; feature count ≠ capability; minimalism ≠ efficiency; protocol mentions ≠ interoperability; permission prompts ≠ sandboxing; a passing command ≠ a proven requirement.
7. Benchmark/quality claims derived from different models, budgets, or setups are not comparable, and every place one is repeated says so.

## 4. Verification protocol and results

Two independent verification layers were applied to the agents' own output:

- **Per-dossier citation spot-checks:** at least 3 citations per dossier re-read against the actual pinned files by the coordinating session (not the subagent that produced them). Result: **26/26 passed** across the six dossiers. The specific citations checked are listed in [research/00-scope.md](research/00-scope.md).
- **Completion gate:** after all deliverables were drafted, the protocol was re-read against the deliverables (gap analysis in [research/12-final-synthesis.md §7](research/12-final-synthesis.md)), and **10 randomly-selected ledger rows** were re-verified against the pinned clones. Result: 9/10 exact; 1 had a line-number offset (the claim was true; the citation was corrected in both the fragment and the merged ledger). Per the gate rule (≤1 failure → no full re-audit), the study closed.

Total: **36/36 citations substantively verified.** The raw per-repo ledger fragments in [research/ledger-fragments/](research/ledger-fragments/) are preserved unmodified as provenance; the merged ledger is [research/11-evidence-ledger.md](research/11-evidence-ledger.md).

Known limitations of the protocol's execution, kept on the record: OpenHands' shipped CLI lives in a third repo that was not pinned (claims about it are marked `UNVERIFIED`); three interoperability pairings are handshake-verified rather than run-verified; the benchmark was designed but not executed. The full list is in [research/12-final-synthesis.md §7](research/12-final-synthesis.md).

## 5. Reproduction

1. Clone the upstream repos and check out the pinned SHAs (table in the [README](README.md)).
2. Pick any ledger row; open the cited path at the cited lines; compare against the claim and snippet.
3. For `external`-class rows, note the fetch date — registries and specs move.
4. For `inference`-class rows, the cited lines are the premises, not the conclusion; disagreement about the inference is a discussion, not a correction.

To challenge a claim, open an issue per [CONTRIBUTING.md](CONTRIBUTING.md) with your own `path:line` citation at the pinned SHA (for errors) or at a newer SHA (for drift reports).

## 6. Changes made for publication

The research files were prepared for publication after the study closed. For the record: working-session status headers (`Status` / `Last updated` / `Next steps`) were replaced with publication headers, with each file's pending next-steps content preserved in place under "Follow-up work"; a handful of findings were re-worded for neutrality without changing any claim, tag, citation, or confidence rating; the prompt file gained a preamble; and the ledger fragments in `research/ledger-fragments/` were left byte-identical to their working-session state. No evidence rows, matrix cells, verdicts, or citations were altered, except as recorded in the completion-gate results above.
