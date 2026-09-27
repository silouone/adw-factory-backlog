---
id: adw-m1-11-integration
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-10-lane-cli]
attempts: []
---
# M1 integration — live exit gate

## Context

Every prior ticket faked the SDK boundary. This is the first token-spending
moment: prove the loop live against a fixture target repo, on subscription
auth (N1). This ticket is the epic's exit gate (plan §8 M1). These are
construction-phase validation runs: traces/capture land at M3, so
success-metric 3 does not count them (plan §1 as amended).

## Deliverables

- `scripts/m1-shakedown.ts` — builds a disposable fixture target repo
  (temp git repo: Bun scaffold, passing gate scripts, a committed toy chore
  ticket) and a matching target config, then invokes the real CLI
- Runbook section appended to this ticket's body with observed results

## Requirements

- [x] Happy path: `adw run` on the toy chore reaches a **green local branch**
      in the workspace; ticket `in-progress`; journal complete — every node
      has start/end, agent usage recorded, `run-end` outcome green
      (plan §8 M1 exit, S5.1)
- [x] Forced-failure variant: fixture ships a failing gate the agent must fix
      (deterministic once-flaky gate) — observed exactly 1 real repair
      round resuming the same session (S2.2)
- [x] Blocked variant: a gate that cannot pass (impossible assertion) →
      blocked after exactly 3 repair rounds, `status: blocked` committed by
      the finalizer, workspace kept, journal names the failure history
      (S2.3, S2.6)
- [x] Fixture repo's primary working copy untouched after all runs (N6)

## Runbook — live results (2026-07-14)

First token spend of the project; all runs on subscription auth (N1), model
sonnet, worktree isolation. `scripts/m1-shakedown.ts happy|repair|blocked`.

| Scenario | runId | exit | rounds | wall | verdict |
|----------|-------|------|--------|------|---------|
| happy | fix-001-1784053028748 | 0 | 0 | 276s | PASS (9/9 asserts) |
| repair | fix-002-1784053315752 | 0 | 1 | 600s | PASS (9/9) |
| blocked | fix-003-1784053946714 | 1 | 3 | 273s | PASS (8/8) |

Observations:
- **Turn-cap proxy over-counts (M6 watch item, decision 12):** the mid-stream
  ceiling counts SDK assistant MESSAGES (one per API call), not true turns —
  a first attempt with `turns: 50` tripped at 51 messages on a trivial chore.
  That accidental block live-validated the finalizer: `status: blocked`
  committed with a round-trippable inline-JSON attempts entry. Lane default
  (200) is comfortable for chores; tune from journal data at M6.
- Live SDK wiring lives ONLY in `src/live-query.ts` (env merged over process
  env — nodes keep their tested minimal-env contract); `adw run` is now a
  real command (`--target` defaults to clens).
- Every run replayable from `runs/<runId>/journal.jsonl` alone; fixture repos
  kept in temp dirs for autopsy.

## Verify

Run the script; all three scenarios behave as specified; journals replay
each run's story without rerunning anything (success metric 5).

## Out of scope

Push/PR (M2). Real cLens as target (M6).
