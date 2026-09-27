---
id: adw-m1-03-status-writer
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-01-ticket-contract]
attempts: []
---
# Status transitions — commit as lock

## Context

Ticket state lives in exactly one place, and every transition is a commit —
git history is the audit log (N3). The `in-progress` commit doubles as the
dispatch lock: a ticket is never dispatched twice (E7, plan §6).

## Deliverables

- `test/intake/status.test.ts` (first, red — uses a temp git fixture repo)
- `src/intake/status.ts` — pure frontmatter rewrite + thin git edge

## Requirements

> Extended by operator gate: `transition` also refuses a dirty ticket file —
> E1 generalized to all transitions. `transition`/`dispatch` take an explicit
> `defaultBranch`; `dispatch` takes an explicit `locksDir` (see status.ts).

- [x] `transition(repoPath, ticketId, newStatus, defaultBranch)` rewrites
      **only** the `status:` line and creates exactly one commit with message
      `adw: <id> → <status>` (N3)
- [x] Dispatch protocol: refuse if the ticket file has uncommitted edits,
      with a descriptive error (E1); require `status: queued` at HEAD; commit
      `in-progress` **before** any provisioning (E7)
- [x] Commits land on the target's default branch in the primary working
      copy, staging **only** `tickets/<id>.md` (other dirty files untouched);
      if the default branch is not checked out there, refuse with a
      descriptive error (plan §5 status-commit mechanics, N6)
- [x] Dispatch check+commit serialized by an exclusive lockfile
      (`<locksDir>/<ticketId>.lock`, `O_EXCL`), released on completion **and on
      every refusal path**; a held lock → descriptive refusal, never a silent
      steal (E7 atomicity)
- [x] Pure rewrite function extracted and unit-tested without git; git called
      directly via subprocess — no wrapper lib (Art. VIII)
- [x] Errors carry `ticketId` + operation (Art. IX)

## Build protocol (Art. I)

1. Fixture helper: `mkTempRepo()` creating a git repo with a committed
   ticket. Tests: happy transition (one commit, only status line changed);
   uncommitted-edit refusal; non-queued-at-HEAD refusal; wrong-branch
   refusal; dirty-but-unrelated files untouched; message format;
   **concurrent-dispatch race** (two simulated invocations — exactly one
   commits `in-progress`, the other refuses).
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

Appending reviewer comments to the body (adw-m2-03), `attempts` appends
(adw-m2-02).
