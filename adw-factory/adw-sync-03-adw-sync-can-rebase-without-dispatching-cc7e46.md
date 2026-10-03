---
id: adw-sync-03-adw-sync-can-rebase-without-dispatching-cc7e46
type: feat
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 120, turns: 250}
depends: [adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696, adw-merge-03-an-unmergeable-pr-is-parked-as-a-draft-2075b4, adw-merge-04-a-local-trial-merge-decides-needs-rebase-25aadb]
attempts: []
---
# `adw sync --rebase` keeps the review queue shippable without dispatching a ticket

> Spec: `specs/adw-v1.17-mergeable-prs.md` (operator-local, NOT in the repo — this ticket is self-contained; do not look for it), D3, stories 15–21. This
> **narrows adw-sync-01**: sync still never dispatches, but with `--rebase` it
> may run the rebase maintenance round.

## Context

The rebase round is wired only into `adw run`'s start-of-run sweep. `adw sync`
passes no rebase hook. A conflicting PR therefore waits until the operator
happens to dispatch another ticket on that target. #174 sat conflicting with
no cmc run after it.

## Requirements

- [ ] **R1** — `adw sync --rebase` passes the rebase hook (never the CI hook)
  to `syncPrState`. `adw sync --rebase --all-targets` does so per target.
- [ ] **R2** — the round's rehydration deps (the gh binding, queries, codex
  auth, hooks, tracing, prompt sink, store, locksDir, runsRoot) are built by
  one shared builder used by both `adw run` and `adw sync`. No second copy.
- [ ] **R3** — `--rebase` with `--dry-run` exits 2 with a message naming both
  flags.
- [ ] **R4** — `adw sync` without `--rebase` is byte-identical in behaviour and
  output to `main` (no agent edge reachable).
- [ ] **R5** — the exit code is 0 unless a target fails to load (existing
  `--all-targets` semantics). `needs-human` and `skipped` outcomes are report
  lines, not failures.
- [ ] **R6** — a `just sync-rebase` recipe (all targets) and the README's
  command reference updated.

## Out of scope

- Scheduling (adw-sync-04).

## Verify

- CLI-seam tests: `--rebase` runs a round on a conflicting fixture;
  `--all-targets --rebase` covers two targets; `--rebase --dry-run` gives
  exit 2; plain sync has no query dependency reachable.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
