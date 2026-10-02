---
id: sabado-55-the-loop-is-driven-from-tests-not-only-through-the-route-977550
type: feat
status: queued
priority: 2
created: 2026-10-02
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: []
attempts: []
---
# test(chat): the loop is driven directly, so the rules that decide a turn stop being untested

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md`, "Test coverage of
> the new loop". Re-verified on `main` at `c82ba2ec`.

## What happens today

Roughly **one line of loop-specific test per ten lines of loop** (+554 in 3 files
against +5 592 of loop and tools). Specifically:

- **`chat_loop.run_turn` and `iter_turn` are never called by any test.** Zero
  hits in `backend/tests/`. The loop is exercised only *through the route*.
- The one multi-round fake (`test_ai_turn_stream.py:32`, `class _Scripted`)
  **clamps at its last element**, so it can never run out. Longest sequence
  scripted: **3 rounds**. `MAX_ROUNDS = 3`, so **no test ever pushes the loop to
  its budget.**
- **`needs_writes` has zero test hits** — the rule deciding whether write tools
  are offered at all is untested.
- **`MAX_ROUNDS` exhaustion is `# pragma: no cover`** by the author's own hand.
- The toolless last round, the nudge `+2` budget, and the drift detector
  (`release_state`, `release_label`, `common_fingerprint`) are all untested.
- `POST /ai/turn/{turn_id}/pending` has **no backend test**; `/undo/{n}` is never
  called in a test.

## Requirements

- [ ] **R1** A fake provider that can script **more rounds than the loop is
      allowed to run** — the current one cannot, and that is why exhaustion is
      untested. It must be able to run out, and the test must see what the loop
      does when it does.
- [ ] **R2** `run_turn` / `iter_turn` are driven **directly**, not only through
      the route. Assert on the **shape of the run** — named invariants over a
      typed transcript — **never on model prose**. (`ai_docs/2026-09-30-go1-agent-architecture-reference.md`
      calls this out as the single highest-leverage testing pattern available
      here, at ~150 lines, and it is why this ticket is `feat` and not `chore`.)
- [ ] **R3** Named invariants, at minimum: the round ceiling holds including the
      nudge's `+2`; the last round is served with no tools; a write never
      precedes its confirmation; a turn that errored still wrote its record.
- [ ] **R4** `needs_writes` is tested as the rule it is — a question that must
      not open writes, an imperative that must, and the mid-turn recovery when
      the nudge fires.
- [ ] **R5** `MAX_ROUNDS` exhaustion loses its `# pragma: no cover`.
- [ ] **R6** Backend tests for `POST /ai/turn/{id}/pending` — the property its
      own docstring calls the security posture (tool and args come off the stored
      row, never the client) is pinned, plus the 404 cross-household branch and
      the 409 no-pending branch. And one for `/undo/{n}`.

## Files

`backend/tests/` only. **No change to `backend/app/`** — if a test cannot be
written without changing the loop, say so in the PR body and stop; do not
refactor production code to make a test convenient.

## Verify

- [ ] `git grep -n 'run_turn\|iter_turn' backend/tests/` returns hits.
- [ ] `git grep -n 'pragma: no cover' backend/app/ai/chat_loop.py` no longer
      covers round exhaustion.
- [ ] Each new test **fails** if the rule it pins is inverted — demonstrate for
      at least R3's four invariants and say so in the PR body.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Cassettes, VCR, respx, or any recorded-provider fixture — none exists today and
this ticket does not introduce one. Any real-provider test. Changing the loop.
