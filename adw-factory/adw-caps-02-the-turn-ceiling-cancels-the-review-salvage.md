---
id: adw-caps-02-the-turn-ceiling-cancels-the-review-salvage
type: bug
status: done
priority: 1
review: false
created: 2026-09-19
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-caps-02-the-turn-ceiling-cancels-the-review-salvage-1789893699380","branch":"adw/adw-caps-02-the-turn-ceiling-cancels-the-review-salvage","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-caps-02-the-turn-ceiling-cancels-the-review-salvage-1789893699380/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/96","provider":"claude","model":"sonnet"}]
---
# A turn breach on a review node throws away work the review lane would have shipped

> Measured 2026-09-19 against run
> `adw-render-03-the-board-becomes-components-1789846487294`. Full postmortem:
> `ai_docs/2026-09-19-render-03-turn-ceiling-cancels-the-salvage.md`.

## The defect

`adw-review-04` established that a review loop which exhausts its rounds with
**green gates** must not throw the run away: the engine's `onExhausted` seam
returns `next`, and the run proceeds to `commit` → `push` → `open-pr`
carrying `ctx.data.reviewExhausted` / `unresolvedFindings` into the PR body.
That is why `adw-render-02` shipped as PR #90 despite a reviewer that never
converged.

The turn ceiling does not participate in that decision. In `engine.ts`'s
retry loop the exhaustion check runs first, and the turn check that follows
returns `abort` unconditionally:

```ts
if (rounds >= maxRounds) {
  ...onExhausted seam — review-04's salvage path...
  return blocked(`node "${node.name}" exhausted ${maxRounds} repair rounds`);
}
if (turnsBreached() && consumesAgentTurns(target)) {
  return abort(`turn ceiling breached: ... refused agent node "${target.name}"`);
}
```

So a ceiling reached **before** rounds are exhausted pre-empts the salvage.
Same lane, same migration, same non-convergence, opposite outcomes:

| run | terminal reason | result |
|---|---|---|
| `adw-render-02` | `review-spec exhausted 2 repair rounds` | **PR #90, merged** |
| `adw-render-03` | `turn ceiling breached: 627/600 — refused "review-fix"` | **nothing** |

## What it cost, once

`adw-render-03` reached **green gates at 503 turns** — lint, typecheck and
test all `pass`, 97 turns under its own cap. The review loop then spent 124
turns re-reading the diff and took the total to 627. The run was aborted
refusing `review-fix` round 2, and **never committed**: 496,535 output tokens
and 107.5 minutes of wall clock, discarded with a complete, gate-green
deliverable sitting in the workspace.

## Not a duplicate of `adw-caps-01`

`adw-caps-01-ceiling-deterministic-tail` protects the **deterministic tail**
from a breach. This breach refused an **agent** node (`review-fix`), which is
the case that check was deliberately written to catch — its own comment cites
caps-01. caps-01's fix as specified would not have saved this run. This is a
third case: a breach on a review node whose gates are already green.

## Requirements

- [ ] **R1 — a turn breach consults the same seam the round ceiling does.**
      When the refused target belongs to a loop that declares `onExhausted`,
      a breach takes that path (`next`) instead of `abort`.
- [ ] **R2 — green gates are the precondition, not the node name.** The rule
      is "gates are green and the only outstanding work is advisory", which
      is what makes review-04's path safe. Do not special-case the string
      `review-fix`.
- [ ] **R3 — the breach is still journalled.** The `turn-ceiling` event still
      fires exactly once. The run reports that it shipped with the ceiling
      breached — the operator must not learn this from the diff.
- [ ] **R4 — every other loop is unchanged.** `gates↻repair`, CI-repair and
      the bug lane's `revise` loop keep today's behaviour exactly, as
      `adw-review-04` already established by setting `onExhausted` nowhere
      else.
- [ ] **R5 — the PR still carries the unresolved findings.** The `open-pr`
      path `adw-review-02` built must not be bypassed by this route.

## Verify

- [ ] Red test first (Art. I): a run whose turn ceiling trips while refusing
      a `review-fix` target, with `gateResults` all-green, reaches `commit`
      rather than `blocked`. RED today.
- [ ] Red test: the same breach against a `repair` target (gates **not**
      green) still aborts. R4's guard.
- [ ] Red test: the shipped PR body contains the unresolved findings and the
      breach is visible in the journal.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- Raising any cap. This ticket is about what a breach *does*, not where the
  line sits.
- The review loop's non-convergence itself — that is
  `adw-review-03-the-reviewer-must-see-its-last-round`.
