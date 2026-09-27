---
id: adw-gates-05-jest-failures-are-named-181b39
type: bug
status: queued
priority: 1
created: 2026-09-27
review: false
caps: {minutes: 90, turns: 400, stallMinutes: 25}
depends: []
attempts: []
---
# A jest gate that fails names no failing test, so the livelock check stops a converging repair loop after two rounds

> Observed 2026-09-27 on `cqc-fe-01-drawer-keeps-keyboard-focus-1790524928011`
> (target `cmc`, provider codex, gates in Docker `node:14.21.3`): the `test`
> gate failed in rounds 2 and 3 with `failingTests: []` both times, so
> `gateFingerprint` produced the identical string `lint:pass|test:fail[]` and
> the engine blocked the run with *"gate outcome repeated across rounds, not
> converging"* — without knowing whether the set of failing tests had changed.
> The repair prompt itself carried the full jest output (9 failed / 580 passed,
> five distinct `●` names), so the agent was not blind; the **factory** was.
> Same defect class as adw-gates-03 (pytest), one runner later. `review: false`:
> hard-gated.

## The defect

`src/pipeline/nodes/gates.ts` `parseFailingTests` is the union of
`parseBunFailingTests` (`(fail) <name>`) and `parsePytestFailingTests`
(`FAILED|ERROR <nodeid>`). Jest prints neither. Its failure lines are

```
  ● Drawer › restores the opener after a backdrop click
  ● Test suite failed to run
```

and, in verbose mode, `  ✕ restores the opener after a backdrop click (12 ms)`.
Both yield `[]`. Two consequences:

1. `gateFingerprint` (engine.ts, adw-cost-00 R2) collapses every jest failure
   set to `test:fail[]`, so ANY two consecutive jest failures look like a
   livelock and the loop stops after 2 of its 3 rounds — even when round 2
   fixed half the tests.
2. `classifyGateOutcome` cannot partition pre-existing from introduced
   (adw-auto-01), so a red baseline on a jest target cannot be subtracted.

## Requirements

- [ ] **R1** A pure `parseJestFailingTests(stdout)`: every `^\s*● (.+)$` line
      whose text is not a jest header (`Test suite failed to run`, `Console`),
      trimmed, in order, deduplicated. The `Test suite failed to run` case is
      kept as the suite path it precedes (the `FAIL <path>` line), so a
      compile error still names something.
- [ ] **R2** `parseFailingTests(stdout)` = bun ∪ pytest ∪ jest, order
      preserved; no behaviour change for bun or pytest output.
- [ ] **R3** The livelock check must never fire on an EMPTY failing-test list
      when the gate's stdout is non-empty: `gateFingerprint` includes a stable
      hash of the test gate's failure tail when `failingTests` is `[]`, so an
      unparsed runner still distinguishes "same failure" from "different
      failure". Journal which branch was taken.

## Verify

- [ ] Red test: the cqc-fe-01 round-2 stdout shape (jest, CI=true) → the five
      `●` names, in order. RED today (`[]`).
- [ ] Red test: a bun fixture + a pytest fixture + a jest fixture concatenated →
      all three sets, bun first.
- [ ] Red test: two gate results with `failingTests: []` and different stdout
      tails → different fingerprints; identical tails → identical.
- [ ] Existing gates/baseline/red-check/engine suites green unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## After it lands

Re-queue `cqc-fe-01` (blocked → queued) in `~/adw/backlog/cmc` and
re-dispatch; its kept workspace shows 609 inserted lines and 9 failing tests
that are the agent's own, so a fresh run is cheaper than a salvage.
