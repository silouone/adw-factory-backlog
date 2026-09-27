---
id: adw-m2-05-operator-surface
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m2
depends: [adw-m1]
attempts: []
---
# Operator surface — run guard, `adw status`, `adw clean`

## Context

The operator is the scheduler (S6.1/S6.2); kept workspaces are reclaimed
only by explicit clean (S4.4, Art. VII). This ticket completes the CLI's
three commands (plan §3).

## Deliverables

- `test/cli-surface.test.ts` (first, red)
- Extensions to `src/cli.ts` (+ small `src/status.ts` if it keeps cli thin)

## Requirements

- [x] Run guard: `adw run` refuses to **dispatch a new build** on a ticket
      already `in-progress` or `in-review`, with a descriptive message naming
      the ticket and its state (S6.2); sync-pr-state maintenance (incl. the
      adw-m2-04 CI round) is exempt — it maintains an existing attempt
- [x] `adw status`: one line per ticket — id, status, priority, attempt
      count, last run outcome + duration from its most recent journal
      (S5.1 reader)
- [x] `adw clean [--ticket <id>]`: the **only** code path invoking
      `workspace.teardown()`; without `--ticket` reclaims all kept
      workspaces; reports exactly what it removed (worktrees, branches)
      (S4.4, Art. VII)
- [x] `clean` never touches workspaces of tickets currently `in-progress` or
      `in-review` (their kept workspace may still serve the CI round,
      adw-m2-04) unless `--ticket` names them explicitly

## Build protocol (Art. I)

1. Temp fixtures with tickets in assorted states + journal files. Tests:
   guard on both states; status output shape; clean's exclusivity
   (grep-level: no other caller of `teardown`) and its in-progress safety.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

Watcher/daemon/parallel runs (spec non-goals).
