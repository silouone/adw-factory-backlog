---
id: sabado-40-a-confirmation-is-redeemed-once-and-expires-f2cefb
type: bug
status: queued
priority: 1
created: 2026-10-01
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-40-a-confirmation-is-redeemed-once-and-expires-f2cefb-1790808576906","branch":"adw/sabado-40-a-confirmation-is-redeemed-once-and-expires-f2cefb","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-40-a-confirmation-is-redeemed-once-and-expires-f2cefb-1790808576906/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# fix(chat): a confirmation executes once and stops being redeemable, so a stale yes cannot fire twice

> ## Base: PR #1283 is merged — this ticket is runnable
>
> `feat/chat-calls-tools` landed on `main` at **2026-10-01T06:01Z**, so `chat_loop.py`,
> `ai_tools*.py`, `usage.py` and `chat_turn.py` are all present. Line numbers below are as
> audited 2026-09-30 on the pre-merge branch and shifted in the rebase — **navigate by
> symbol, not by line**.
>
> **A first attempt was fired at 2026-09-30T22:49Z, seven hours BEFORE that merge**, cut
> from `29ae7af5` where none of these files existed. All four such runs blocked on
> `red-check exhausted` having produced nothing — an agent cannot write a failing test for
> a defect in a file that is not in its tree. The `attempts` entry below is that run. It is
> not a verdict on this ticket.

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 5 (d) — one of the
> three `SEC:` items in the PR's second required-fix clause.

## What happens today

`POST /ai/turn/{turn_id}/pending` (`ai.py` ≈`:753-781`) loads the `ChatTurn`, checks
`turn_row.user_id != user.id` → 404, takes `proposed_actions[0]`, and calls
`redeem(db, user, action["tool"], action["arguments"])`.

**The security posture is sound and worth preserving:** the client sends only
`class PendingAnswer(BaseModel): confirmed: bool` (`ai.py` ≈`:744-745`). Tool and
arguments come off the owner-scoped row, never from the request. An endpoint taking a tool
name from the client would be an endpoint for running any write on the account.

**Two defects sit beside it:**

1. **No idempotency.** Nothing marks `proposed_actions` resolved. `redeem` runs the
   executor past the gate every time. **Two POSTs = two executions** — so a double-click,
   a retried request, or a replayed one deletes twice, or creates twice.
2. **No expiry.** There is no age check on the turn. **A pending from last month is still
   redeemable** — a person who abandoned a confirmation weeks ago can have it fire by
   replaying the request.

`POST /ai/turn/{turn_id}/undo/{n}` (`ai.py` ≈`:790-819`) has the same shape and the same
gap.

## Requirements

- [ ] **R1** A redemption marks its action resolved, atomically with executing it. A second
      POST for the same action returns a **409**, not a second execution.
- [ ] **R2** A pending older than a bounded window is refused — **410 Gone** (it existed
      and no longer does) rather than 404, which would be indistinguishable from someone
      else's turn. Pick the window and state the reasoning in the ticket's PR body; the
      audit suggests minutes, not days.
- [ ] **R3** `/undo/{n}` gets the same treatment: redeem-once, and an age bound.
- [ ] **R4** The 404-on-foreign-turn behaviour is **preserved exactly**. The repo's own
      rule (`chat_feedback.py` ≈`:17-18`) is that a turn the caller does not own answers
      404 rather than 403 *"so the reply never confirms that someone else's turn exists"* —
      and that is better than the reference system, which returns 403. Do not regress it.
- [ ] **R5** The client still sends only a boolean. Do not widen `PendingAnswer`.
- [ ] **R6** A refusal is legible to the person: an expired or already-redeemed
      confirmation says so in the thread rather than failing silently.

## Files

`backend/app/api/ai.py` (the `/pending` and `/undo/{n}` routes) ·
`backend/app/models/chat_turn.py` (whatever records resolution — a column or a field in
`proposed_actions`) · one Alembic migration **if** a column is added ·
`frontend/src/api/ports/ai.ts` + the conversation component for R6 ·
`backend/tests/`. Nothing else.

## Verify

- [ ] **The red test, first.** POST `/pending` twice with `{confirmed: true}` for the same
      turn; assert the write executed **once** and the second call returned 409. It must
      **fail on the current tree** — today both execute.
- [ ] A second red test: a turn older than the window returns 410 and does not execute.
- [ ] A third: `/undo/{n}` called twice undoes once.
- [ ] **Regression:** a foreign turn still returns **404, not 403** (R4). Assert it
      explicitly — this property is currently **untested on the backend**, which is how it
      would be lost.
- [ ] Regression: the happy path — one POST, one write, the thread updates.
- [ ] If a migration is added: `alembic heads` shows one head, and the chain reads
      correctly by `revision`/`down_revision`, not by filename prefix (the branch carries
      duplicated numeric prefixes).
- [ ] Target gates all green.

## Out of scope

Widening confirmation to the ten unconfirmed writes (`sabado-39`). The metering bypass
(`sabado-41`). An idempotency key on `POST /ai/turn` itself — a real gap, its own ticket.
