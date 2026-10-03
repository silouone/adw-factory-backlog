---
id: sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996
type: feat
status: blocked
priority: 1
created: 2026-10-02
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [sabado-29-every-pr-walks-a-real-browser-through-a-real-backend-a98ee9]
attempts: [{"runId":"sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996-1791012492958","branch":"adw/sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996-1791012492958/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# test(e2e): a conversation is walked in a real browser on every PR, so the chat stops being the one surface nothing protects

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §2, the
> boxed quote; re-anchored in `ai_docs/2026-10-02-sabado-chat-entrypoint-state.md` §3.
> **This ticket is OVERDUE, not pending.** The synthesis said it *"must exist
> before 1283 merges, or nothing protects it"*. PR #1283 merged
> **2026-10-01T06:01Z**. It is the single highest-priority unfiled item in the lot.

## What happens today

Fourteen Playwright journeys run on every PR — register, session, onboarding,
documents, vault, circles, calendar ×3, biens, foyer, authenticated-routes, plus
two bundle specs (`sabado-29`, `sabado-30`, `sabado-31`, all `done`).

Grep every one of them for `chat`, `conversation` or `/ai/`:

```
→ nothing
```

**The e2e work protected every surface except the one that just absorbed ~8% of
the codebase.** `git merge` will never show you this, and no unit test will
either: `chat_loop.run_turn` and `iter_turn` are **never called by any backend
test** (`ai_docs/2026-09-30-sabado-pr1283-seven-axes.md`, "Test coverage of the
new loop"), and the suite contains **zero `data-testid`** — every assertion is a
French accessible name or a CSS class, so it is blind by construction to a UI it
never named.

## Requirements

- [ ] **R1** One new journey spec, in the same harness and the same style as the
      existing fourteen (read `sabado-29`'s spec and its helpers first; reuse the
      throwaway-account fixture from `sabado-28`, do **not** invent a second
      account lifecycle).
- [ ] **R2** The journey walks, in one browser session, against a real backend:
      ask a question → the tool chain renders its steps → an answer arrives →
      the answer can be rated → a **deletion** is proposed and the confirmation
      modal is confirmed.
- [ ] **R3** Assertions are on **observable user-facing state**, never on model
      prose. The chain's step labels come from `ai_tools.DOING`'s closed
      vocabulary and its outcomes from `outcome()` — assert those, and assert the
      *shape* of the run (a step appears `loading` then `done` under the same
      `n`), not the sentence the model wrote.
- [ ] **R4** The journey is **deterministic enough to gate a PR**. State plainly
      in the PR body how model non-determinism is handled — a scripted/fake
      provider, a fixed seed question whose tool choice is forced, or a retry
      budget. A flaky gate on the chat is worse than none.
- [ ] **R5** It runs in the same CI job as the other journeys, on every PR. No
      new workflow, no opt-in flag.

## Files

`frontend/e2e/` (the journey + any helper it needs) · the CI workflow **only** if
the existing job cannot pick the spec up by glob. Nothing in `backend/app/`.

## Verify

- [ ] `git grep -nE 'chat|conversation|/ai/' frontend/e2e/` returns the new spec.
- [ ] The journey fails if `POST /ai/turn` 500s, and fails if the confirmation
      modal never appears — demonstrate both by temporary local break, and say so
      in the PR body.
- [ ] Target gates: `frontend-check` · `frontend-build` · `test-e2e-boot` ·
      `test` · `test-frontend` — all green.

## Out of scope

Adding `data-testid` across the existing fourteen journeys. Any change to
`chat_loop.py` or the tool catalogue. Rating-data persistence beyond what
`chat_feedback.py` already does.
