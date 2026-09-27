---
id: adw-m2-03-sync-pr-state
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m2
depends: [adw-m1]
attempts: []
---
# sync-pr-state — ticket follows the PR lifecycle

## Context

Humans at the two ends (Art. IV): the factory never merges — it only
*observes* what the human did and moves the ticket accordingly, at the start
of every invocation, before ticket selection (plan §6 drift mitigation).

## Deliverables

- `test/pipeline/nodes/sync-pr-state.test.ts` (first, red — fake `gh`)
- `src/pipeline/nodes/sync-pr-state.ts` + CLI wiring (runs before selection)

## Requirements

- [x] For every ticket `in-review` with a recorded PR: query state via
      `gh pr view --json` (faked in tests)
- [x] Merged → transition `done` (S3.2)
- [x] Closed without merge → transition `rejected` and append the reviewer's
      PR comments to the ticket body under a dated `## Review feedback`
      section — context for a future run. **No automatic re-queue**: re-entry
      to `queued` is an operator edit only (S3.3 as amended, plan §5); each
      write its own commit (N3)
- [x] Runs unconditionally at the start of `adw run`, before selection
      (plan §6) — wired in `runTicket` (syncPrState before selection); ordering
      proven by `test/cli.sync-pr-state.test.ts`
- [x] **Negative capability test:** grep-level assertion that the codebase
      contains no `gh pr merge`, `gh pr review --approve`, or `gh pr close`
      invocation anywhere (S3.1, Art. IV)

## Build protocol (Art. I)

1. Fake `gh` returning merged / closed / still-open; temp git fixture for
   ticket mutations; ordering test (sync before selection); the grep test.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

CI checks themselves (adw-m2-04 hooks in here); re-queueing rejected tickets
(operator edit — normal selection picks them up afterwards).
