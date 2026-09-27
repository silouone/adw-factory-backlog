---
id: adw-m2
type: epic
status: done
priority: 1
created: 2026-07-14
depends: [adw-m1]
children:
  - adw-m2-01-push-node
  - adw-m2-02-open-pr
  - adw-m2-03-sync-pr-state
  - adw-m2-04-ci-round
  - adw-m2-05-operator-surface
  - adw-m2-06-integration
attempts: []
---
# EPIC M2 — Lifecycle complete

From green local branch to merged PR and back: push, PR open with a body
rendered from journal stats, PR-lifecycle sync (merged → done, closed →
rejected + reopen), the single CI repair round, and the operator surface
(run guard, `adw status`, `adw clean`).

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m2-01-push-node | push, gate-green guard, bounded retries | S2.7 E6 N6 |
| adw-m2-02-open-pr | PR body from journal + `gh pr create` + in-review | S1.5 S1.6 S5.4 |
| adw-m2-03-sync-pr-state | merged→done, closed→rejected+comments | S3.1–S3.3 |
| adw-m2-04-ci-round | ≤1 CI repair round, next-run via sync | S2.4 E8 |
| adw-m2-05-operator-surface | run guard, `adw status`, `adw clean` | S6.2 S4.4 |
| adw-m2-06-integration | scratch-repo lifecycle exit gate | plan §8 M2 exit |

## Exit criteria

Toy chore → real PR on a scratch GitHub repo → operator merges → next
invocation transitions the ticket to `done`; the close-without-merge path
lands in `rejected` with reviewer comments appended (operator re-queues by
edit). Ticket file's git history shows every transition as its own commit.
