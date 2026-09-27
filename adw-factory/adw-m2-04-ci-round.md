---
id: adw-m2-04-ci-round
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m2
depends: [adw-m2-02-open-pr, adw-m2-03-sync-pr-state]
attempts: []
---
# CI repair round — at most one, on a later invocation

## Context

Remote CI is confirmation, not the primary gate (decision 7): local gates
already passed. Per the plan §3 node graph, CI repair happens on a **later
invocation**, triggered from sync-pr-state (`[next run] ci-fail→repair`) —
the factory does not sit waiting for CI. State must therefore survive the
process: sessionId from the journal, attempt branch + runId from the
ticket's `attempts:`, and the kept workspace (teardown is clean-only,
Art. VII). One round absorbs both real failures and flakes — v1 deliberately
does not distinguish them (E8).

## Deliverables

- `test/pipeline/nodes/ci-round.test.ts` (first, red — fake `gh`)
- `src/pipeline/nodes/ci-round.ts` + wiring inside sync-pr-state

## Requirements

- [x] Triggered from sync-pr-state when an `in-review` ticket's PR has
      failing checks (`gh pr checks`, faked in tests) (S2.4, plan §3 graph)
- [x] Rehydrates prior state: sessionId from the run's journal, attempt
      branch + workspace path via the `attempts:` runId; missing or cleaned
      workspace → `blocked` as a state error, **no round charged** (E3
      flavor)
- [x] Fetch failing log (`gh run view --log-failed`), repair via
      `query({resume: sessionId})`, re-run **local gates** in the kept
      workspace, re-push — at most **1** round; the round count is persisted
      in the `attempts:` entry so it survives restarts (S2.4)
- [x] Second CI failure → `blocked` via the finalizer, with full transcript
      + failure history attached, same shape as local blocking (S2.4, S2.3)
- [x] CI pass (first look or after the round) → ticket stays `in-review`,
      journal records the round count (S5.1)
- [x] Run-guard interplay: CI repair via sync is maintenance of an existing
      attempt, not a new dispatch — S6.2's refusal applies to new builds
      only (asserted in tests)
- [x] Flake needs no special handling: the single round covers it (E8)

## Build protocol (Art. I)

1. Fake `gh` sequences: pass; fail→pass; fail→fail; checks still pending.
   Fixtures with persisted attempts/journal state. Assert rehydration, round
   accounting across two simulated invocations, session resume,
   local-gates-before-repush, blocked shape, no-round-charged on missing
   workspace.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

Flake detection/statistics (explicit non-goal, E8); waiting for CI within
the dispatching run.
