---
id: adw-m9
type: epic
status: done
priority: 1
created: 2026-09-12
depends: []
children:
  - adw-m9-01-review-verdict-contract
  - adw-m9-02-review-node
  - adw-m9-03-two-axis-stages
  - adw-m9-04-review-fix-loop
  - adw-m9-05-verdict-in-pr-body
  - adw-m9-06-lane-wiring
attempts: []
---
# EPIC M9 — the agent reviewer

`specs/adw-v1.3-review-lane.md`. A read-only two-axis reviewer between green
gates and commit, whose findings can send work back to a fresh builder and
which always brief the human in the PR body.

**Why:** the three lanes have no stage that JUDGES. v1.1's Residual Risks names
three semantic holes (trivially-passing tests, a repair round weakening the
bug's test, clean-red being structural not semantic) and assigns every one to
"operator review at merge". The two-axis `/code-review` has gated seven
milestone exits by hand. This moves a proven manual step inside the machine.

**Amends** `adw-v1-plan.md` §2 Gate III: 2 agent node types → 3 (`review`
joins `build` and `repair-resume`). The fix stage reuses `build`, so only one
type is added.

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m9-01-review-verdict-contract | The typed verdict: types + pure parser | R3 |
| adw-m9-02-review-node | The `review` node type, write-denied | R1 R4 |
| adw-m9-03-two-axis-stages | Standards + Spec instances, parallel | R2 |
| adw-m9-04-review-fix-loop | Floor → fresh build-fix, bounded | R5 D1 D7 |
| adw-m9-05-verdict-in-pr-body | Findings reach the human | R6 |
| adw-m9-06-lane-wiring | All three lanes, per-ticket opt-out | R7 R8 |

## Exit criteria

A run with a deliberately-seeded defect has the reviewer name it, route it to a
fresh fix stage, and land the remaining findings in the PR body — journaled
end-to-end. Plus the §6 metric that decides whether this was worth it: operator
review minutes per PR fall. If they do not, revert it.

## Out of scope

A merge gate; a quality score; replacing `gates` or Story 3's human review.
