---
id: sabado-60-a-turn-has-a-wall-clock-434498
type: feat
status: queued
priority: 2
created: 2026-10-02
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: [sabado-54-a-transient-provider-failure-retries-the-round-not-the-turn-c3f9bb]
attempts: []
---
# feat(chat): a turn has a wall clock, so one question cannot hold a worker for twenty-five minutes

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 4 (d), the
> "Runaway bounds" bullet, and (e) third bullet. Re-verified on `main` at
> `c82ba2ec`.

## What happens today

Nothing bounds a turn in time. `_MISTRAL_TIMEOUT` is **300 s per call**, the
nudge grants `budget + 2` so the true ceiling is **5 rounds**, and nothing caps
calls per round at all (`for call in requested:`, `chat_loop.py:852`, runs all 40
if the model returns 40).

**Worst case: 5 × 300 s = 25 minutes of held worker on one question**, with the
person's browser long gone — and `AI_TURN_STREAM.md:148-150` is explicit that
*"a dropped connection means the turn is lost"*, so that worker is holding a turn
nobody will ever read.

> ⚠️ **Two bounds compose wrong in this repo already.** `MAX_ROUNDS = 3` plus the
> announce guard's `+2` produced errored turns at rounds 4, 5 and **6**
> (`ai_docs/2026-09-30-sabado-chat-state.md` §3). The standing rule from that
> finding: **one owner per bound, and the owner is the loop, not the feature.**
> This ticket adds a third bound. It must be designed knowing about the other two.

## Requirements

- [ ] **R1** A wall-clock deadline for the whole turn, owned by the loop,
      checked at a round boundary. State the value and its derivation in the PR
      body — it should be defensible against the measured p95 (5.33 s) and the
      300 s per-call timeout, not picked round.
- [ ] **R2** **The three bounds compose, provably.** A test asserts that no
      combination of the round ceiling, the announce guard's `+2` and the retry
      from `sabado-54` can carry a turn past the deadline. This requirement is
      the reason the ticket exists; a deadline that another bound can cancel is
      the bug, not the fix.
- [ ] **R3** A turn stopped by the deadline **keeps what it wrote**. It ends the
      way an exhausted round budget ends — an answer the person can read, the
      turn record written, the spend booked. It is not an exception.
- [ ] **R4** The deadline trip is visible in the turn record, distinguishable
      from an exhausted round budget and from a provider error.
- [ ] **R5** No change to `MAX_ROUNDS`, the nudge, or the per-call timeout.

## Files

`backend/app/ai/chat_loop.py` · `backend/app/api/ai.py` (only if the deadline
needs to reach the streaming body) · `backend/tests/`.

## Verify

- [ ] A test drives the loop with a slow scripted provider past the deadline and
      asserts the turn ends with a readable answer and a booked spend.
- [ ] The composition test in R2 exists and is green.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

A per-round or per-turn **call** cap — real, named in the same audit, and not
this ticket. Resumability, SSE `id:`/`Last-Event-ID`, idempotency keys. Any
change to the weekly allowance.
