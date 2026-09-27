---
id: adw-bug-17-a-live-baseline-holder-is-never-stale
type: bug
status: done
priority: 1
created: 2026-09-20
review: false
caps: {minutes: 90, turns: 400}
depends: []
attempts: [{"runId":"adw-bug-17-a-live-baseline-holder-is-never-stale-1789857851437","branch":"adw/adw-bug-17-a-live-baseline-holder-is-never-stale","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-17-a-live-baseline-holder-is-never-stale-1789857851437/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/93","provider":"claude","model":"sonnet"}]
---
# The baseline lock is declared stale after a fixed 20 minutes, so a slow target's waiters all measure their own baseline at once

> Observed 2026-09-19, seven sabado runs fired together. `sabado-11` took
> `runs/baselines/f33c38c8….lock` at 23:00 and was still measuring (two full
> gate passes, ~36 min on that target) when, at 23:20, every waiter crossed
> `BASELINE_LOCK_STALE_MS` and started its own measurement: seven baselines,
> fourteen gate passes, load average 116 on 12 cores, every run killed by
> hand. The lock's whole purpose — one measurement per base commit, adopted
> by the rest — was defeated by the target being slower than cLens.
> `review: false`: hard-gated.

## The defect

`src/pipeline/nodes/baseline.ts`: `BASELINE_LOCK_STALE_MS = 20 * 60_000`;
`waitOnLock` computes `deadline = holder.acquiredAt + BASELINE_LOCK_STALE_MS`
and breaks the lock past it. Staleness is a clock, not a fact about the
holder. The lock record already carries the holder's `pid` and `runId`;
neither is consulted.

## Requirements

- [ ] **R1** A lock is stale only when its holder is **not alive**: the
      recorded `pid` no longer exists (`process.kill(pid, 0)` throws
      `ESRCH`), or the holder's run has journaled `run-end`. A live holder
      is waited on indefinitely by default.
- [ ] **R2** The waiter re-checks liveness on every poll tick, journaling
      one `baseline` `wait` event per minute with the holder's `runId` and
      elapsed time, so a long wait is visible in `just watch`.
- [ ] **R3** The wall-clock bound stays as a backstop, raised to
      `BASELINE_LOCK_STALE_MS = 3 h`, and is journaled as a distinct reason
      (`stale-by-clock` vs `stale-holder-dead`) when it fires.
- [ ] **R4** Pure liveness predicate, injectable (`isAlive(pid)`), so the
      tests need no real processes.

## Verify

- [ ] Red test: a lock whose holder is alive and 25 minutes old is **not**
      broken; the waiter adopts the cache when it appears. RED today.
- [ ] Red test: a lock whose holder pid is dead is broken immediately,
      reason `stale-holder-dead`.
- [ ] Red test: a live holder past 3 h is broken with reason `stale-by-clock`.
- [ ] Existing baseline suite green unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
