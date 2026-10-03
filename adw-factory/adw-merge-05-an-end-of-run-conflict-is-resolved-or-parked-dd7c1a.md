---
id: adw-merge-05-an-end-of-run-conflict-is-resolved-or-parked-dd7c1a
type: feat
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 180, turns: 300, stallMinutes: 25}
depends: [adw-merge-03-an-unmergeable-pr-is-parked-as-a-draft-2075b4]
attempts: []
---
# An end-of-run rebase conflict is resolved before push, or the PR opens parked

> Spec: `specs/adw-v1.17-mergeable-prs.md`, D1, stories 1–9. This **lifts
> adw-merge-01 D3** ("a conflict in the lane spawns no agent").

## Evidence

Every lane conflict so far shipped a conflicting PR. `rebase-onto-base`
journaled `rebase-conflict` in 6 runs, and all 6 ended `green` with a ready,
conflicting PR the operator then rebased by hand: cqc-be-30, cqc-be-35,
sabado-38, cqc-fe-45, cqc-fe-37 and cqc-fe-39 (#174). Clean lane rebases
(31 of 31) are fine and must not change.

## Requirements

- [ ] **R1 — resolve in the lane.** On a lane conflict, instead of aborting:
  run one **fresh** resolve session with the rebase round's resolve prompt
  (ticket body, plan, conflicted hunks), then the round's continue logic
  (marker check, `git rebase --continue`, re-conflict detection), then the
  gates on the rebased commit, then push. Reuse the round's nodes or
  builders; do not copy them (Art. VIII).
- [ ] **R2 — budget.** The lane resolve is capped at 1 session, recorded on
  the attempt as its own field (not `rebaseRounds`). The maintenance round's
  own cap of 1 is unchanged.
- [ ] **R3 — fallback.** If the resolve fails, markers remain, a later commit
  re-conflicts, or the post-resolve gates are red, the lane aborts or
  resets to the pre-rebase commit (the `before` sha it already records),
  pushes that, and `open-pr` opens a **draft** carrying the adw-merge-03
  label. The PR body gains a section listing the conflicted paths and which
  step failed. The run's outcome stays `green`. A `pr-parked` event is
  journaled.
- [ ] **R4 — clean path untouched.** A clean rebase, and a clean rebase with
  red gates (still a hard stop: nothing pushed), behave exactly as on `main`.
- [ ] **R5 — all three lanes.** chore, feat and bug share the tail, so all
  three get it. Each lane file's header chain diagram is updated.
- [ ] **R6 — journaled like every agent node.** node-start/end, prompt,
  usage and capture for the resolve session.

## Out of scope

- Non-worktree isolation (the node already no-ops there, D1 of adw-merge-01).
- Any change to the maintenance round.

## Verify

- Lane-seam tests (`runLane` with the real lane spec, real git fixture, fake
  agent): conflict plus a good resolve gives a rebased, gated, pushed, ready
  PR; conflict plus a bad resolve gives a reset, the original commit pushed,
  a draft with the label and body section, and outcome green; post-resolve
  gates red gives the same fallback; a clean rebase is unchanged.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
