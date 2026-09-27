---
id: adw-bug-21-a-blocked-run-hides-why-it-blocked
type: bug
status: done
priority: 1
created: 2026-09-26
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-bug-21-a-blocked-run-hides-why-it-blocked-1790463578740","branch":"adw/adw-bug-21-a-blocked-run-hides-why-it-blocked","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-21-a-blocked-run-hides-why-it-blocked-1790463578740/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/112","provider":"claude","model":"sonnet"}]
---
# A blocked run hides why it blocked: the board and run screen show "blocked" and nothing else

> **Evidence, 2026-09-26, run `adw-store-01-tickets-dir-1790448708493`.**
> The build node failed after 91 min with
> `agent reported an error result: API Error: 400 messages.947: The final block in an assistant message cannot be 'thinking'`.
> The journal carries it twice: `node-end {node:"build", outcome:"fail", reason}`
> and `run-end {outcome:"blocked", reason}`. The web UI shows neither. The
> card and run header only say `blocked`. 25 changed files (+1,941) sit
> uncommitted in the worktree, and nothing on screen says so. The operator
> found out by reading the CLI log.

## What happens today

`src/web/projection.ts` reads `run-end` for `outcome` and `durationMs` only
and drops `reason`. The Gantt renders a `watchdog` reason on a block but
never a failed node's `node-end.reason`. The board card's outcome
vocabulary (`green/blocked/warn/live`) has no slot for a cause.

## Requirements

- [x] **R1 — the reason reaches the view model.** `RunView` carries the
      `run-end.reason` when present, and the failing node's name plus its
      `node-end.reason`. They are never fabricated; absent stays absent.
- [x] **R2 — the board card** of a blocked run shows a one-line cause:
      failing node plus the reason, truncated, with the full text on
      hover/focus. The same component and wording are used on the run screen
      (`adw-fe-15`).
- [x] **R3 — the run screen** shows the full reason in the header area, and
      the failed node's Gantt block is marked failed with its reason in the
      node drawer.
- [x] **R4 — "work left behind".** When a blocked run's workspace still has
      uncommitted changes, the run screen says so, with the file count and
      the workspace path, so the operator knows there is something to
      salvage. The count is computed server-side from the workspace; a
      missing workspace renders nothing, not zero.
- [x] **R5 — tests.** Projection tests: `run-end` with and without a reason;
      a failed `node-end` with a reason; a watchdog block unchanged. Render
      tests: the card cause line, the header reason, the drawer reason, and
      the work-left-behind line (present, absent, missing workspace).

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`. Manually, `adw web` on
`adw-store-01-tickets-dir-1790448708493`: the card and run screen show the
API 400 reason and "25 uncommitted files left in the workspace".

## Blocked by

None. This ticket can start immediately.
