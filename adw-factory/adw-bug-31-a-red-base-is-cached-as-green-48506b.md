---
id: adw-bug-31-a-red-base-is-cached-as-green-48506b
type: bug
status: done
priority: 1
created: 2026-09-30
caps: {minutes: 180, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-bug-31-a-red-base-is-cached-as-green-48506b-1790802751200","branch":"adw/adw-bug-31-a-red-base-is-cached-as-green-48506b","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-31-a-red-base-is-cached-as-green-48506b-1790802751200/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/166","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A red base is cached as green whenever the test runner writes its results to a report file

Measured 2026-09-30 on target `cmc`, base `cqc/release-1` @ `edad125`. The base's test gate
fails deterministically: one test, every run, 0.7 s in isolation on a quiet machine. Yet
`runs/baselines/edad125a45f09a4197ceb4055853101ded779c01.json` records:

```json
{"sha":"edad125…","ok":true,"runs":2,"flaky":[],
 "gates":[{"name":"lint","outcome":"pass"},{"name":"test","outcome":"pass"},{"name":"build","outcome":"pass"}]}
```

Because the base looked green, the red-base=blocked rule never fired. Three CMC runs
(cqc-fe-21, -22, -26) each burned 3 repair rounds on a failure that no ticket had introduced:
about $5.80 and 1 h 50 min, and all three ended `blocked`.

## Two defects compose

1. **The baseline doesn't read the declared report.** `src/pipeline/nodes/baseline.ts:472`
   runs `failingTests: parseFailingTests(exec.stdout)`, which reads stdout only and ignores
   `gate.report`. The `gates` node was taught to read reports (adw-gates-05/07) at
   `gates.ts:671`, which uses `reportFailingTests ?? parseFailingTests(exec.stdout,
   exec.stderr)`; the baseline never got that. CMC's test gate runs jest with
   `--json --outputFile=.adw/report.json`, so stdout carries no test names. Every failing
   baseline pass yields `failingTests: []`.
2. **"All runs failed, no names" reconciles to `pass`.** `reconcileBaselineRuns`
   (`baseline.ts:~586-592`) sets `reproducing = intersectAll([[], []]) = []`, then
   `reproducing.length > 0 ? fail : pass`, which gives **`pass`**. `flakyNames` is also
   empty, so nothing at all is recorded. A gate that exited non-zero in every pass is cached
   as a clean pass.

Either defect alone would be caught. Together, any target whose test runner writes a report
file gets a green baseline on a red base.

## Red first (Art. I)

- **Pure:** `reconcileBaselineRuns([testGate], [[{name:"test",outcome:"fail",failingTests:[]}],
  [{name:"test",outcome:"fail",failingTests:[]}]])` must **not** return `outcome: "pass"`.
  It passes today.
- **Node:** with a fake workspace whose test-gate exec exits 1 with empty stdout and a jest
  report on disk naming one failure, the baseline must record that failure by name. Today it
  records a pass.

## Acceptance criteria

- [ ] The baseline reads failing tests the same way `gates` does: prefer `gate.report`
      (delete it before each pass, read it after a failing exec), fall back to stdout **and**
      stderr. Share one helper between the two nodes rather than keeping a second copy.
- [ ] A test gate that fails in **every** pass is never recorded as `pass`. With no nameable
      failures it is `outcome: "fail"` with no `failingTests`. That means "red, unattributable",
      so the base-green check blocks, and it says in its reason that the names could not be
      read.
- [ ] Cache records carry evidence for every gate: exit code and, for the test gate, the
      report's total/failed counts when one exists. A record without it counts as a miss
      (bump the cache schema so every existing record, including `edad125…`, is re-measured).
- [ ] Journal the reconcile decision per gate (`reproducing`, `flaky`, `unattributable`) so
      a false green can never again be invisible, as it was here.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
