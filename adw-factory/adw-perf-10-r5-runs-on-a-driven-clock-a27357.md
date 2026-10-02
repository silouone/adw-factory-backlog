---
id: adw-perf-10-r5-runs-on-a-driven-clock-a27357
type: chore
status: in-review
priority: 2
created: 2026-10-02
depends: []
attempts: [{"runId":"adw-perf-10-r5-runs-on-a-driven-clock-a27357-1790899243209","branch":"adw/adw-perf-10-r5-runs-on-a-driven-clock-a27357","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-10-r5-runs-on-a-driven-clock-a27357-1790899243209/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/170","provider":"claude","model":"claude-sonnet-5-5","rebased":"d1ecb7378c7844cf583be903b6d0648fc6356507","ciRounds":1}]
---
# One web-server test waits 70 real seconds; the clock it needs is already injectable

`test/web/server.test.ts:2831`, "R5 (adw-bug-29): holding a real connection open ≥60s against a
finished run sends exactly ONE data frame", measured **70.2 s**. It is the slowest single test in the suite
(`ai_docs/2026-10-02-test-suite-time-audit.md`).

The test holds a real SSE connection open until `Date.now() + 61_000` (`:2839`), on the default real
`pollTimer` and clock. The test sets its own 75 s timeout (`:2849`).

`createWebServer` already accepts `clock` (`src/web/server.ts:245`), `pollTimer` (`:251`) and `serve`
(`:240`). Heartbeats are judged against `clock()` (`:900-906`, `:1034`).

## What to change (test only, no src change)

- `pollTimer`: a timer that fires every ~20 ms of real time.
- `clock`: advances 1 s of simulated time per tick, for 61 ticks.
- `serve`: `(o) => Bun.serve({ ...o, idleTimeout: 1 })`. The connection then still crosses the real idle-timeout cutoff several times. R5 is the only test that checks this on a real socket; R3 at `:3632` only pins the value 30.
- Assertion unchanged: exactly one data frame.

## Acceptance criteria

- [ ] R5 under 2 s in isolation. It still fails if a finished run emits a second data frame. Prove this by temporarily breaking the guard, and say so in the PR.
- [ ] Its custom 75 s timeout is removed.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.
