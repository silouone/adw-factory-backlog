---
id: adw-perf-07-a-green-baseline-runs-once-b341d3
type: feat
status: done
priority: 1
created: 2026-10-01
depends: []
attempts: [{"runId":"adw-perf-07-a-green-baseline-runs-once-b341d3-1790892320853","branch":"adw/adw-perf-07-a-green-baseline-runs-once-b341d3","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-07-a-green-baseline-runs-once-b341d3-1790892320853/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"adw-perf-07-a-green-baseline-runs-once-b341d3-1790892320853","branch":"adw/adw-perf-07-a-green-baseline-runs-once-b341d3","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-07-a-green-baseline-runs-once-b341d3-1790892320853/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/202","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A green baseline runs once: the second pass only exists to classify a red one

Measured 2026-10-01 on `adw-target-01-silou-hq-is-a-target` (a one-file chore). The `baseline` node
alone took about 15 minutes before any agent started.

`runReproducingGates` runs the whole gate list `BASE_RUN_COUNT` (2) times, **unconditionally**. On
adw-factory that means the 5207-test suite twice. The second pass exists to tell a reproducing
failure from a flaky one (adw-auto-03). When pass 1 is green, there is nothing to classify, and the
second pass is pure cost: 6–7 minutes per new base sha on adw-factory, and 2–3 minutes on cmc.

## What to build

- Pass 2 runs **only if pass 1 has a failing gate**. A fully green pass 1 is the baseline, recorded with `runs: 1`.
- The cache records how many passes ran. A cached single-pass green is a valid hit.
- Optional, by measurement: a red pass 1 re-runs **only the failing gates**, not the whole list.

## Red first (Art. I)

- With a fake workspace whose pass 1 is all green, `exec` is called once per gate, not twice, and the outcome is green with `runs: 1`.
- With pass 1 red, a second pass runs and reconciliation is unchanged (the adw-auto-03, adw-bug-31 and adw-bug-35 tests keep passing).

## Acceptance criteria

- [ ] On adw-factory, a fresh green base computes in one suite run (time in the PR).
- [ ] Spec amendment if adw-auto-03's text says "always two passes" (Amendment rule).
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing). It pairs with adw-bug-35 (reconciliation) and adw-bug-36 (slow rebase tests).
