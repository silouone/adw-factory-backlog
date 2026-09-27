---
id: adw-m6
type: epic
status: done
priority: 2
created: 2026-07-14
depends: [adw-m3]
children:
  - adw-m6-01-shakedown-runs
  - adw-m6-02-metrics-tuning
  - adw-m6-03-review-fixes
  - adw-m6-04-capture-completeness
attempts: []
---
# EPIC M6 — Shakedown on real work

Prove the factory on genuine cLens chores from its live backlog, measure the
spec's success metrics from journals (verified, not assumed), tighten
defaults from data, declare v1 done. Children are coarse (plan §8) —
**refine each at pickup**.

Reordered before M4/M5 (2026-07-14 post-review): shakedown runs on
**worktree isolation** — real-customer proof before isolation expansion
(Art. II; spec S4.5: v1 ships worktree, container/remote are "delivered
next"). M4/M5 are post-v1 expansion.

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m6-01-shakedown-runs | ≥3 real cLens chores through the lane, merged | S7.1 S7.2, metric 1 |
| adw-m6-02-metrics-tuning | metrics from journals + tightened defaults | metrics 2–5, decision 12 |
| adw-m6-03-review-fixes | adversarial-review fixes (M2/M3/M6) | milestone external review |

## Exit criteria

≥ 3 real cLens chores merged with zero human-written code; all success
metrics measured and reported; v1 declared done.
