---
id: adw-bug-41-an-approval-outage-costs-a-run-not-a-wait-4b1e07
type: bug
status: done
priority: 1
created: 2026-10-03
caps: {minutes: 120, turns: 300}
depends: []
attempts: [{"runId":"adw-bug-41-an-approval-outage-costs-a-run-not-a-wait-4b1e07-1790989655951","branch":"adw/adw-bug-41-an-approval-outage-costs-a-run-not-a-wait-4b1e07","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-41-an-approval-outage-costs-a-run-not-a-wait-4b1e07-1790989655951/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/188","provider":"claude","model":"claude-sonnet-5-5"}]
---
# An approval outage still costs a whole run: the retry doesn't wait, and the ticket lands `blocked`

## Root cause update (2026-10-03, after filing)

The "outage" was **ours**. The factory spawns the Claude Code binary bundled in
`node_modules/@anthropic-ai/claude-agent-sdk-darwin-arm64`. It was **2.1.209**
(SDK 0.3.209, installed 2026-09-15), while `package.json` and `bun.lock` had pinned
0.3.287 since #182. The same probe, run at the same minute, was **denied** by
bundled 2.1.209 every time ("claude-sonnet-5-5 is temporarily unavailable") and
**approved** by CLI 2.1.288 and by bundled 2.1.287 after
`bun install --frozen-lockfile`. Only commands that need the classifier fail.
Read-only compound commands are auto-approved, which made it look intermittent.

So **item 0 below is the P1 fix**. Items 1–3 still hold, since a real outage would
burn runs the same way, but they are secondary.

**0. Preflight: refuse to dispatch on a stale install.** Before dispatch,
`adw run` compares the installed `@anthropic-ai/claude-agent-sdk` version
(`node_modules/.../package.json`) with the version `bun.lock` resolves. On a
mismatch it refuses with exit 2 and names both versions and the fix
(`bun install --frozen-lockfile`). The comparison is a pure function over the two
parsed strings. `just doctor` reports the same check. Tests: match → proceed;
mismatch → exit 2 with both versions in the message; a missing install → exit 2.

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
