---
id: adw-bug-41-an-approval-outage-costs-a-run-not-a-wait-4b1e07
type: bug
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 120, turns: 300}
depends: []
attempts: []
---
# An approval outage still costs a whole run: the retry doesn't wait, and the ticket lands `blocked`

## Why

adw-bug-40 (#181) stopped agents from coding blind during an auto-mode
classifier outage. The run is still lost, though. Overnight 2026-10-02/03,
**8 of 10 claude build runs** ended `blocked` with `tool approval unavailable`:
hq-12 ×2, hq-13, hq-19, sabado-55, sabado-56, sabado-61 and adw-gates-10. The
sabado-39 CI round was lost the same way. Full evidence is in
`ai_docs/2026-10-03-last-20-runs-failure-report.md` (gitignored, on the operator's machine).

Two causes, both measured from the journals:

1. **The retry doesn't wait.** `makeTransientRetryNode()`
   (`src/pipeline/nodes/build.ts`, near `TRANSIENT_RETRY_MAX_ROUNDS = 2`) is a
   no-op `{kind: "next"}`. Gaps between a run's successive `approval-outage`
   events were 13–28 s for hq-12, hq-13, hq-19 and sabado-56, so all 3 attempts
   were spent in under a minute. The outage windows lasted 10–45 min
   (21:14–21:26, 00:15–00:59).
2. **Exhaustion lands `blocked`.** The ticket then needs a hand requeue, or a
   hand salvage: hq-12, adw-gates-10, hq-13 and hq-19 were all finished code
   (silou-hq PR #15, #16, #17; adw-factory PR #185). A longer backoff alone is
   not enough: sabado-55 was denied at 00:28, 00:44 and 00:59, so it waited
   31 min and was still blocked.

## What to build

1. **Wait before an outage retry.** When the retried node's last result carried
   `approvalOutage`, the transient-retry step waits before re-spawning. Use an
   injected clock and sleep, the same seam the engine already injects
   (`setTimeout`/`clearTimeout` defaults in `engine.ts`). The schedule is a pure,
   exported function of the round number, and the total wait stays inside the
   run's remaining wall-clock cap. A plain transient retry with no outage keeps
   today's zero wait.
2. **An exhausted outage requeues instead of blocking.** When the rounds run out
   with `approvalOutage` set, the run ends with a distinct outcome. The CLI
   returns the ticket to `queued`, the same way `requeueDeferred`
   (`src/cli.ts`) does, but **records an attempt** whose outcome names the outage
   and keeps the workspace path. Unlike adw-bug-33's deferred case, the agent did
   start and may have left finished work, so the attempt must point at it for
   salvage. Journal the `run-end` reason as today, so `adw web` still shows the
   real cause.
3. **A CI round blocked by the outage writes its `node-end` and `run-end`.** It
   leaves the ticket `in-review` and doesn't charge `ciRounds`. Measured on
   sabado-39 `…-ci-1790979251950`: no terminal events, ticket flipped to
   `blocked`, round charged.

Out of scope: resuming a run in its old workspace (adw-resume-01, which awaits
operator approval of its amendment), and reducing classifier dependence through
the allowlist (a separate measurement).

## Tests first (Art. I)

- The pure wait schedule: round → ms. Zero for a non-outage retry. Never longer
  than the remaining cap.
- The engine with a fake clock: an outage retry waits the scheduled time before
  the re-spawn. The fake clock advances, and no real time passes.
- An exhausted outage: the outcome is distinct from `blocked`, the CLI returns
  the ticket to `queued` with one attempt recorded, and the workspace is kept.
- A CI round in an outage: the ticket stays `in-review`, `ciRounds` is unchanged,
  and `run-end` is present.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, all green.
