---
id: adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696
type: feat
status: done
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: [adw-bug-42-a-rebase-round-never-vanishes-mid-session-b782a6]
attempts: [{"runId":"adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696-1791033525527","branch":"adw/adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696-1791033525527/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/198","provider":"claude","model":"claude-sonnet-5-5","rebased":"110a720d5eeede9315aea86cb73ec2911c445380"}]
---
# A rebase round holds the ticket's lock, and a crashed round is reconciled

> Spec: `specs/adw-v1.17-mergeable-prs.md` (operator-local, NOT in the repo — this ticket is self-contained; do not look for it), D5 and D6, stories 28–33.

## Context

`rebaseRound` takes no lock (its header says "no dispatch lock is taken").
On 2026-10-02 three `adw run`s on cmc started within 36 s of each other. Each
one runs the PR-state sweep, so each can rebase the same kept workspace. Once
a scheduled sweep exists (adw-sync-04), that overlap becomes routine. A round
killed hard leaves the workspace mid-rebase and its journal with no end
(evidence in adw-bug-42).

## Requirements

- [ ] **R1 — one lock namespace.** A round acquires the same per-ticket
  `O_EXCL` lock file that dispatch uses, in the same `locksDir`, for its whole
  duration, and releases it on every exit path. First check and record in the
  PR whether the dispatch lock is still held while a ticket is `in-review`.
  If it is, stop and propose an amendment rather than diverging.
- [ ] **R2 — owner record.** The lock file contains `{pid, runId, startedAt}`
  as JSON. The dispatch path writes the same record (existing callers keep
  working).
- [ ] **R3 — live owner means skip.** A sweep that finds the lock held by a
  live pid returns `skipped` with reason `locked by <runId>`. No git, no gh
  write, no charge.
- [ ] **R4 — dead owner means reclaim.** A lock whose pid is not alive (or an
  empty legacy lock older than the round's caps) is reclaimed. The reclaim is
  journaled in the new round's journal with the dead owner's runId.
- [ ] **R5 — crash reconciliation.** Before probing, the round checks the kept
  workspace for a rebase in progress. If one is found, the round aborts it
  and appends a synthesized end to the orphaned rebase run's journal (the
  `synthesized-exit` precedent; reason `reconciled: owner gone`). It then
  returns `needs-human` without charging. The original charge stays (no refund).
- [ ] **R6** — `adw run`'s embedded sweep and `adw sync` honour the same lock.

## Out of scope

- Parking the PR as draft after reconciliation is wired in by adw-merge-03.
- Any change to dispatch-lock semantics beyond the owner record.

## Verify

- Tests at the sync seam: live-pid lock means skipped; dead-pid lock means
  reclaimed and journaled; mid-rebase workspace means aborted, the orphan
  journal gets a synthesized end, and no charge; two concurrent
  `syncPrState` calls on one ticket produce exactly one round.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
