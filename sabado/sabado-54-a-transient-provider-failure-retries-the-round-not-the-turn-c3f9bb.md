---
id: sabado-54-a-transient-provider-failure-retries-the-round-not-the-turn-c3f9bb
type: bug
status: queued
priority: 2
created: 2026-10-02
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# fix(chat): a transient provider failure costs a round, not the whole turn the person already paid for

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 7 (d), the
> "Retries/backoff/timeouts" bullet. Re-verified on `main` at `c82ba2ec`.

## What happens today

The loop has **no retry**. A provider exception anywhere inside it is caught once,
at the bottom:

```python
# backend/app/ai/chat_loop.py:982
except Exception as exc:  # noqa: BLE001 — a turn that already wrote must not lose its report
```

and becomes `error`. No round is retried. So **a single transient 503 on round 2
ends the turn — and round 1's tokens have already been booked against the
person's weekly allowance.** The SDK's own retries are left in place for the HTTP
call (`provider.py:311-345`), but a failure that escapes them kills the turn.

The catch itself is correct and must survive: a turn that already wrote must keep
its report. This ticket adds a retry **above** it, not a replacement for it.

## The red test

A scripted provider raises a transient error on its second call and succeeds on
the third. Assert the turn completes with an answer. Today it returns `error` —
that is the red.

## Requirements

- [ ] **R1** A failed round is retried with backoff, bounded — state the bound
      and justify it in the PR body. **One owner per bound, and the owner is the
      loop** (`ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §7: `MAX_ROUNDS`
      plus the announce guard's `+2` already compose wrong and reached round 6).
      A retry must **not** consume round budget, and must **not** be able to push
      the turn past its ceiling.
- [ ] **R2** Only *transient* failures retry — timeouts, 429, 5xx. A 4xx that is
      not 429, a refusal, or a malformed tool call is not transient and still
      ends the round as it does today.
- [ ] **R3** A retried round does **not** re-execute a tool that already ran. The
      repeat-call memo (`chat_loop.py:790`) exists; reuse it rather than adding a
      second idea of "already done".
- [ ] **R4** The existing bottom catch at `:982` stays, unchanged in intent: a
      turn that already wrote keeps its report.
- [ ] **R5** A retry is visible — in the turn record, so the bench can tell a
      slow turn from a retried one.

## Files

`backend/app/ai/chat_loop.py` · `backend/tests/`.

## Verify

- [ ] The red test above is green.
- [ ] A test asserts a non-transient failure does **not** retry.
- [ ] A test asserts retries cannot carry a turn past its round ceiling.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

A wall-clock deadline for the turn — that is
`sabado-60-a-turn-has-a-wall-clock-434498`, and the two bounds must be designed
knowing about each other. Provider fallback. Changing `_MISTRAL_TIMEOUT`.
