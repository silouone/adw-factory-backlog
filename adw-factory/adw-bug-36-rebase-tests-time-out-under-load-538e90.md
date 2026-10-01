---
id: adw-bug-36-rebase-tests-time-out-under-load-538e90
type: bug
status: queued
priority: 2
created: 2026-10-01
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# The rebase tests spend three minutes on real git and time out when runs overlap

`test/pipeline/nodes/rebase.test.ts` and `rebase-round.test.ts` (added by adw-merge-01) take
**182 s for 69 tests** on a clean checkout of `6b058da`. They pass in isolation, but drive real git.

Inside the full suite (5207 tests), with other runs and their baselines going at once, 17 of them
exceed the 30 s per-test timeout. On 2026-10-01 that turned a green base into a "red" cached
baseline (with adw-bug-35) and blocked three runs. It also inflates every baseline: two full-suite
passes took ~35 minutes under load.

## The fix (pick by measurement)

- Build each scenario's repositories once from a template (`git clone --local` of a prepared fixture, or one bare repo per file), instead of `git init` plus commits per test.
- Where real git is not the point, use the scripted-git seam the file already has ("scripted git failures").
- Keep the few real-git cases that prove real behaviour, but make them fast. Raise their own per-test timeout only if they genuinely need it, never the suite's.
- **Added 2026-10-02 (test audit).** Build the template fixture as **one shared helper next to `test/real-git-workspace.ts`**, not a per-file copy. adw-perf-09 reuses it for ci-round and sync-pr-state, which have the same fixture shape. While you're in that file, stop running every command through `sh -c` (`:23`); it costs two processes per git call. Measured spawns: rebase-round 892 git calls for 28 tests, rebase 542 for 41.

## Red first

- A timing guard: both files together finish under a stated budget (e.g. 30 s) on the operator's machine, measured and written into the test file's header. No test there may exceed 5 s in isolation.

## Acceptance criteria

- [ ] Same 69 behaviours covered (count unchanged or justified).
- [ ] The full `bun run test` shows no rebase test in the slowest 10.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
