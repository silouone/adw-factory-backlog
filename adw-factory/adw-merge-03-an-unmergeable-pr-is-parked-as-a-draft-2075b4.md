---
id: adw-merge-03-an-unmergeable-pr-is-parked-as-a-draft-2075b4
type: feat
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: [adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696]
attempts: []
---
# A PR the factory can't make mergeable is parked as a draft, not blocked

> Spec: `specs/adw-v1.17-mergeable-prs.md` (operator-local, NOT in the repo — this ticket is self-contained; do not look for it), D7, stories 35–39 and 42. The
> operator chose "open as draft + label" on 2026-10-03.

## Context

Today a failed rebase round calls `finalizeBlocked`, so the ticket leaves
`in-review`. The PR stays open and ready, so it still sits in the operator's
review queue, conflicting. And because the sweep only reads `in-review`
tickets, the operator's later hand-merge never moves the ticket to `done`.

## Requirements

- [ ] **R1 — park.** A round that fails for any reason except lease-refused
  or skip-prefixed, or that finds its resolve budget spent, or that
  reconciled a crash (adw-merge-02 R5), runs `gh pr ready --undo`, then
  `gh pr edit --add-label adw:needs-rebase` (creating the label if the repo
  lacks it), and returns a new `needs-human` sync outcome carrying the reason.
  The ticket stays `in-review`; `finalizeBlocked` is no longer called by the
  round.
- [ ] **R2 — unpark.** A green round on a PR that is draft and carries the
  label runs `gh pr ready` and removes the label.
- [ ] **R3 — ownership guard.** Every park and unpark gh write first checks
  that the PR's head branch equals the attempt's branch. On a mismatch it
  writes nothing and returns `skipped` with the reason.
- [ ] **R4 — journaled.** Each park and unpark appends a journal event
  (`pr-parked` / `pr-unparked`) with the reason, in the round's journal.
- [ ] **R5 — report.** The sync report renders `needs-human` distinctly
  (one line, with the reason).
- [ ] **R6** — the label name is one exported constant, shared with
  adw-merge-05 and adw-web-03.

## Out of scope

- The lane's own draft PR (adw-merge-05).
- Web chips (adw-web-03).

## Verify

- Sync-seam tests with fake gh asserting the exact argv: failed round, then
  park calls and the ticket still `in-review`; green round on a parked PR,
  then ready and label removed; head-branch mismatch, then zero writes;
  a later MERGED reading moves the parked ticket to `done`.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
