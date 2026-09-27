---
id: adw-learn-05-fence-a-held-out-eval-set-0ad553
type: manual
status: queued
priority: 2
created: 2026-09-28
depends: [adw-learn-02-a-run-names-the-factory-that-produced-it-3ceb3e]
attempts: []
---
# Fence a held-out eval set the factory is never tuned against

> `adw-learn-*` group (`ai_docs/2026-09-28-prompt-training-readiness.md` §3.5).
> `type: manual`: the pick is the operator's, and the factory cannot pause for
> a pick (memory: HITL design tickets are built in-session). The code half,
> a manifest and its checker, is small and is built here under Art. I once
> the pick is made.

## Why this exists

Every prompt or hook change so far has been judged on the next live runs,
which are also the runs that motivated the change. That is tuning on the
training set. Before `adw-learn-06` replays a single stage, the tickets it
is scored on must be fixed, recoverable and off-limits to tuning.

## The operator's half

- [ ] Pick **15 to 20 tickets** from the banked runs, across the three lanes
      and at least two targets, each satisfying all of:
      - the run reached a terminal outcome, green or blocked, with a full
        journal;
      - `baseline.sha` still exists in the target repo;
      - the ticket file at dispatch time is recoverable from the ledger's git
        history at the run's `dispatch` commit;
      - for feat and bug lanes, `artifacts/plan.md` is present;
      - for bug lane, `build-test-only`'s red-check verdict is journaled.
- [ ] Aim for roughly half green and half blocked, so the set can detect a
      regression as well as an improvement.
- [ ] Record the pick in `eval/holdout.json` in the code repo.

## The code half

- [ ] **R1** `eval/holdout.json`: an array of
      `{ ticketId, runId, target, lane, baseSha, ticketCommit, outcome }`.
      Committed. Never edited by a run.
- [ ] **R2** `scripts/holdout-check.ts`: for each entry, verify every
      recoverability condition above against the disk and the two git repos,
      and print one line per entry with a pass or the first failing condition.
      Exit non-zero on any failure. Read-only.
- [ ] **R3** A test asserts no ticket id in `eval/holdout.json` appears in
      any `attempts[]` entry created after the manifest's commit date, other
      than by `adw-learn-06`'s replay harness (which tags its runs). This is
      the fence.
- [ ] **R4** `tickets/README.md` gains a paragraph: what the hold-out is, and
      that a prompt or hook change is reported against it, never tuned on it.

## Verify

- [ ] `bun scripts/holdout-check.ts` passes on every entry the day it is
      committed.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Running anything against the set: that is `adw-learn-06`.
