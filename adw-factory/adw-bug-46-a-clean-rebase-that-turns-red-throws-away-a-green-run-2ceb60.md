---
id: adw-bug-46-a-clean-rebase-that-turns-red-throws-away-a-green-run-2ceb60
type: bug
status: queued
priority: 1
created: 2026-10-03
depends: [adw-merge-03-an-unmergeable-pr-is-parked-as-a-draft-2075b4, adw-merge-05-an-end-of-run-conflict-is-resolved-or-parked-dd7c1a]
---
# A clean rebase that turns the gates red throws away a green run

> Spec: `specs/adw-v1.17-mergeable-prs.md` (operator-local). This ticket
> **reverses adw-merge-05 R4** ("a clean rebase with red gates is still a hard
> stop: nothing pushed"). It needs an operator decision before dispatch.

## Evidence

**1. Lane: hq-35 (silou-hq), run `hq-35-today-shows-perso-and-pro-calendars-fbb29b-1791033481915`.**
It ran 17 min ($2.31) and passed gates, review-standards and review-spec. Then
`rebase-onto-base` replayed cleanly, but a sibling PR had added a required
`rebuild` to `ActionDeps`. `gates-rebased` failed typecheck twice (TS2741 in
two new test fixtures), and the run ended `blocked` with nothing pushed. The
fix was two `rebuild: async () => {}` lines. It was salvaged by hand as
silou-hq PR #37.

**2. Rebase round: sabado-61, run `sabado-61-ingestion-commits-per-batch-9ed89f-rebase-1791031864784`.**
The round rebased PR #1362, then `gates` failed `test` with
`failingTests: []`. The run ended:

```
node "gates" exhausted 0 repair rounds
```

The ticket went to `blocked`, so the sweep stopped reading it. The operator
merged #1362 at 14:21Z with CI green, and the ticket stayed `blocked` until a
hand fix. Three defects:
- The round's gates node is the plain `makeGatesNode`. adw-bug-43's re-run
  once reached only the lane's `gates-rebased`, so one load-flaky pass is
  still terminal in a round.
- `maxRounds: 0` is deliberate (src/pipeline/nodes/rebase-round.ts), so the
  "exhausted 0 repair rounds" reason is accurate but says nothing: no gate
  output, no test names.
- A round that blocks calls `finalizeBlocked`, which is the split
  adw-merge-03 R1 already removes.

## Requirements

- [ ] **R1, the lane parks instead of discarding.** When `gates-rebased` is red
      after its re-run, the lane resets to the pre-rebase commit (whose gates
      were green), pushes it, and opens a **draft** with the adw-merge-03
      label. The PR body names the red gate and its evidence (adw-bug-43 R1).
      The run's outcome is `green`, and a `pr-parked` event is journaled. This
      reuses adw-merge-05 R3's fallback path rather than copying it (Art. VIII).
- [ ] **R2, the round re-runs once.** The rebase round's gates use the same
      re-run-once node as `gates-rebased` (adw-bug-43 R2), with the same
      evidence in its reason.
- [ ] **R3, the round parks.** A round still red after its re-run parks the PR
      (adw-merge-03 R1) instead of blocking the ticket.
- [ ] **R4, the amendment.** v1.17's merge-05 R4 line is amended to match R1.

## Out of scope

A repair agent after the rebase. Parking keeps the work, and the operator or
the next round fixes it with real evidence.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`
