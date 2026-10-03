---
id: adw-bug-43-one-red-pass-after-rebase-throws-away-the-run-27bcc8
type: bug
status: in-review
priority: 1
created: 2026-10-03
caps: {minutes: 120, turns: 300}
depends: []
attempts: [{"branch":"adw-bug-43-one-red-pass-after-rebase-throws-away-the-run-27bcc8","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/192","provider":"claude","model":"claude-opus-5-5","note":"built in-session","runId":"hand-built-in-session","workspace":"in-session"}]
---
# One red gate pass after the rebase throws away the whole run, and nothing says what failed

## Evidence

`sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996-1791012492958`
ran for 3.1 h. Its gates were green, then base moved (#1358 merged) and
`rebase-onto-base` replayed the commit. `gates-rebased` ran once, `test` went red,
and the run ended `blocked` with only:

> gates-rebased: gates red on the rebased commit — gate "test" failed; nothing is pushed

The journal has no gate output and no failing test names: a `fail` result carries
no `gates` rider, so the engine journals nothing. The baseline cache shows that
sabado's `test` suite is load-flaky. The 4 `test_household_read.py` tests that
"reproduced" on 68929f8 pass 28/28 in isolation on both 68929f8 and d0b86085.

## Root cause

`makeRebasedGatesNode` (src/pipeline/nodes/rebase.ts) runs one stop-at-first
gates pass and turns any `retry` into a hard `fail`, keeping only the gate name.
That is the same pattern as adw-bug-13 Defect B (one flake is terminal), on a
node that adw-bug-13 did not touch.

## Requirements

- [x] **R1, evidence.** When gates-rebased fails, its reason names the gate, exit
      code, failing test names (when parsed) and a bounded output tail. The same
      evidence is journaled as a `rebased-gates` event through the `onRebase` sink.
- [x] **R2, one re-run.** A red pass re-runs the failing gate and every gate after
      it (they never ran, because the pass stops at the first failure), exactly
      once. If the re-run is green, the node proceeds and a `rebased-gates` event
      with `verdict: "flaky"` carries the first pass's evidence. If the re-run is
      red, the node hard-stops as today with `verdict: "red"` and both passes'
      evidence.
- [x] Gates that passed before the failing one are not re-run. `fix` is still
      never applied, and no repair loop is added.
- [x] A spec amendment in `adw-v1.1-lanes.md` records the re-run (Art. VI: the
      spec says what the node does).

## Verify

- [x] Red tests in `test/pipeline/nodes/rebase.test.ts` cover R1 and R2 (a flaky
      gate is green on the second pass, a red gate is red twice, earlier gates run
      once).
- [x] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
