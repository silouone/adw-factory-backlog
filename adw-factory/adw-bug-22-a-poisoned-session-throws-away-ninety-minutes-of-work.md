---
id: adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work
type: bug
status: done
priority: 1
created: 2026-09-26
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work-1790470050008","branch":"adw/adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work-1790470050008/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work-1790472773032","branch":"adw/adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work-1790472773032/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/117","provider":"claude","model":"sonnet"}]
---
# A poisoned session throws away ninety minutes of work instead of resuming from the checkpoint

> **Evidence, 2026-09-26, run `adw-store-01-tickets-dir-1790448708493`**
> (Claude Code CLI 2.1.209, `claude-sonnet-5`, effort `high`). After 473
> assistant messages the build died with
> `API Error: 400 messages.947: The final block in an assistant message cannot be 'thinking'`.

## Why it happened (from the session transcript)

The model's response `…Zg1oVNKh` streamed `thinking > tool_use`. The CLI
executed the tool, and then the **same response** emitted one more `thinking`
block after the tool call. The CLI immediately sent the next request (the
trailing thinking block, the tool result and the error share one timestamp,
20:54:29Z). That replay held an assistant message whose final block was
`thinking`, which the API rejects. Across the whole session this shape
occurred **once in 473 messages**, only on that message; every other message
ends in `tool_use` or `text`.

This is a CLI/API edge case the factory cannot prevent. It is also
**unrecoverable in place**: resuming the same session replays the same
invalid history and fails the same way. Today the node fails, the run ends
`blocked`, and the work is abandoned in the worktree. That work covered
R1–R6 and R8 of the ticket, `test-fast` was green (2,103 tests), and the
checkpoint was current.

## The fix: treat it as a poisoned session and hop

The factory already has the right mechanism. `runCappedHops` (`adw-cost-02`)
starts a **fresh session seeded from `.adw/artifacts/<node>-checkpoint.md`**
and never resumes the old session.

## Requirements

- [x] **R1 — classify.** A pure classifier recognises a *poisoned-session*
      agent error: an API 400 whose message names the conversation
      structure (`messages.N: …`, including "final block … cannot be
      `thinking`"). It is distinct from transient errors (which retry) and
      from real failures. The classifier is table-tested against the exact
      string above plus negative cases.
- [x] **R2 — hop, don't die.** In a `capAndResume` node, a poisoned-session
      error triggers the same fresh-session hop as the turn cap: read the
      checkpoint, start a new session with the "RESUMED SESSION" preamble,
      and keep the worktree as it is. It counts against the same hop and
      run-level budgets. No checkpoint means fail as today, with a reason
      that says the session was poisoned and no checkpoint existed.
- [x] **R3 — bounded.** At most N poisoned-session hops per node (N=2),
      then fail with a reason naming the count.
- [x] **R4 — journaled.** Each such hop journals an event with the node, the
      API error text and the hop number, so the run screen (see
      `adw-bug-21`) can show it.
- [x] **R5 — tests.** Drive `runCappedHops` with a fake stream that returns
      the poisoned-session error once: the next hop starts a fresh session
      (no `resume`), its prompt carries the checkpoint, and the node then
      succeeds. Also cover: poisoned twice past the bound fails; no
      checkpoint fails with the descriptive reason.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`.

## Blocked by

None. This ticket can start immediately. It pairs with
`adw-bug-23-the-turn-cap-never-counts-a-claude-turn`, which makes sessions
hop before they grow large enough to make this rarer failure more likely.
