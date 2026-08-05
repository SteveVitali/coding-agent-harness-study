# Contributing

This repository is a snapshot study: every claim is pinned to the commit SHAs listed in the [README](README.md). Contributions that improve its accuracy are welcome. The same evidence rules the study held itself to apply to corrections.

## Citation corrections (errors at the pin)

If a cited `path:line` does not support its claim **at the pinned SHA**, open an issue using the *Citation correction* template with:

- The file and claim being challenged (research file + line, or ledger row).
- The correct `path:line` citation at the pinned SHA, with a ≤3-line verbatim snippet.
- What you believe the corrected claim should say (or that it should be withdrawn).

Verified corrections are committed on top of the `v1.0-snapshot-2026-08-04` tag, so the original record stays intact and the diff is the errata trail.

## Drift reports (stale findings)

If upstream has changed since the pin and a finding no longer holds at HEAD, open an issue labeled `drift` with the newer SHA and citation. Drift reports don't make the study wrong — every claim is scoped to its pin — but they are valuable context for readers, and enough of them may justify a re-pin study.

## What is not in scope

- Re-litigating a project's quality without `path:line` evidence. Opinions about the subjects belong on their own trackers.
- Benchmark results. [research/10-benchmark-design.md](research/10-benchmark-design.md) is a design; if you *run* it, open an issue — results with methodology would be genuinely interesting, but they'll be linked rather than merged, since this repo's claims are static-analysis-only.
- Adding new subject projects. Out of scope for this snapshot; a future study may widen the cohort.

## Typos and broken links

PRs welcome directly, no issue needed.
