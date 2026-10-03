---
id: adw-bug-42-a-rebase-round-never-vanishes-mid-session-b782a6
type: bug
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A rebase round never vanishes mid-session: it always ends with a run-end and a clean workspace

> Spec: `specs/adw-v1.17-mergeable-prs.md`, story 34. It blocks the schedule
> (adw-sync-04), because polling multiplies whatever makes a round disappear.

## Evidence

The only production rebase round, `runs/cqc-fe-45-the-list-fits-its-data-a43230-rebase-1790975695169`
(2026-10-02, cmc target, provider **codex**, model gpt-6-sol):

- Its journal goes `run-start`, then `rebase` (rebase-conflict on
  `CheckedContentSummary.tsx`), then `rebase-charge` (next; the charge was
  committed to the ticket store as `rebaseRounds: 1`), then `rebase-resolve`
  node-start, prompt, and one `heartbeat` 15 s later. **Nothing after that:**
  no node-end, no `run-end`.
- The kept workspace was left **mid-rebase** (reflog: `rebase (start)` at
  23:14:55). The operator finished it by hand at 23:47 (`rebase (continue)`).
- `cqc-fe-44`'s run-start is timestamped **27 ms** after the round's last
  heartbeat. That points to the same `adw run` process having moved on from
  its sweep to dispatch while the round's engine never journaled an end.
  This is not proven: an external kill and relaunch would look similar.

## Requirements

- [ ] **R1 — reproduce first (Art. I).** A test at the sweep seam
  (`syncPrState` with the real `rebaseRound`, fake gh, real git fixture with a
  conflict, codex provider with a fake codex query) that reproduces "the round
  returns or the sweep proceeds without a `run-end` in the rebase journal".
  Try the candidate causes in order: the codex resolve path settling without
  a node-end; a promise in the round or sweep not awaited; a throw escaping
  `runLane`. If none reproduces, write that up in the ticket's PR with the
  evidence and land R2 alone.
- [ ] **R2 — invariant.** Whatever the cause, every rebase round that wrote a
  `run-start` writes a `run-end`, and a round that did not finish green leaves
  the kept workspace with **no rebase in progress** (aborted). Asserted for:
  the agent throws, the agent query resolves early, and an abort signal.
- [ ] **R3** — fix the cause R1 found, red to green.

## Out of scope

- Crash recovery after a hard kill (SIGKILL) is adw-merge-02.
- Refunding the charge.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test` green.
- The R1 test is red on `main` before the fix (when a cause reproduces).
