---
id: sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a
type: bug
status: queued
priority: 2
created: 2026-10-01
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a-1790808592086","branch":"adw/sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a-1790808592086/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a-1790842816351","branch":"adw/sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-41-the-weekly-allowance-cannot-be-bypassed-2a070a-1790842816351/workspace","outcome":"blocked"}]
---
# fix(chat): the weekly allowance cannot be bypassed by the old routes, and OCR is inside the gate

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

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 7 (d) — the remaining
> two `SEC:` items of the PR's second clause.

## What happens today

**The allowance itself is the best-executed axis in the PR, and this ticket does not touch
it.** `ensure_allowance` runs **before the first model call** (`ai.py` ≈`:718`) and
raises 429; `record_turn` books the whole turn as one
`INSERT … ON CONFLICT DO UPDATE` whose `CASE` **replaces** a stale week's counter rather
than adding to it — so two concurrent turns cannot both read the same stale total, and no
Monday cron exists to fail to run. Keep all of that.

**Two holes sit around it.**

1. **The old routes bypass it entirely.** `POST /ai/chat` (`ai.py` ≈`:382-414`) calls
   `provider.chat` and returns — **no `record_turn`, no `ensure_allowance`**, only a
   30-per-5-min request-count limit. `POST /ai/analyze` likewise (20 per 5 min). The
   front no longer calls `/ai/chat` (only `/ai/analyze`, `/ai/execute` and `/ai/turn`
   survive in `frontend/src/api/ports/ai.ts`) — but **both routes stay registered**, so an
   authenticated client whose weekly allowance is exhausted can keep spending by calling
   `/ai/chat` directly.
2. **OCR spend is outside both the gate and the meter.** `_read_the_page` fires a vision
   `analyze` call at `ai.py` ≈`:696`; `ensure_allowance` is at ≈`:718`. So **a blocked
   account can still burn vision calls by uploading a scan**, and `record_turn` books only
   `observation["tokens"]` from the loop, so those vision tokens never reach the weekly
   counter. They do reach `cost.note_call` — which is a ContextVar **no HTTP request
   opens**, so they go nowhere.

**Note the third caller.** `backend/app/whatsapp/service.py` also calls `provider.chat`
and is likewise unmetered. It is **not** in this ticket's file list — the WhatsApp path is
its own ticket — but decide deliberately whether R1 deletes a route WhatsApp still needs.

## Requirements

- [ ] **R1** `/ai/chat` and `/ai/analyze` either **enforce the allowance** or **cease to
      exist**. State which you chose and why in the PR body. Deleting is cleaner if nothing
      calls them — **establish that first**, including the WhatsApp path and any
      integration.
- [ ] **R2** `ensure_allowance` runs **before** the vision OCR call, not after it.
- [ ] **R3** The vision call's tokens are added to `record_turn`, so a turn that read a
      scan reports what it actually spent.
- [ ] **R4** The allowance mechanism is otherwise untouched: the pre-loop check, the
      single-statement booking with its stale-week `CASE`, the never-interrupt-mid-turn
      rule, and the hand-written `weekly_limit` override all stay exactly as they are.
- [ ] **R5** A blocked account gets the same 429 whichever door it knocks on.

## Files

`backend/app/api/ai.py` (the two old routes, the OCR-call ordering, `record_turn`) ·
`backend/app/api/usage.py` **only if** the booking signature must widen ·
`frontend/src/api/ports/ai.ts` **only if** a route is deleted ·
`backend/tests/`. Nothing else.

## Verify

- [ ] **The red test, first.** An account whose allowance is exhausted calls
      `POST /ai/chat` directly and asserts **429**. It must **fail on the current tree** —
      today it answers and spends.
- [ ] A second red test: an exhausted account uploads a scan and asserts **no vision call
      was made** (R2). Today the call fires before the gate.
- [ ] A third: a turn that read a scan reports prompt+completion tokens **including** the
      vision call (R3).
- [ ] Regression: the pre-loop 429, the stale-week replacement, and a normal turn's booking
      are all unchanged. Assert the `CASE` behaviour directly — two turns in the same week
      after a stale row.
- [ ] If R1 deletes a route: `git grep` proves no caller remains, **WhatsApp included**,
      and the PR body says so.
- [ ] Target gates all green.

## Out of scope

Opening `cost.metering()` for HTTP requests so `cost.note_call` stops writing into a
closed scope — a real and cheap fix, its own ticket. Prompt caching (`sabado-4x`). The
WhatsApp path's own metering.
