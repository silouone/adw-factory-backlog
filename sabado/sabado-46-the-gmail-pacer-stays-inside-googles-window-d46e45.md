---
id: sabado-46-the-gmail-pacer-stays-inside-googles-window-d46e45
type: chore
status: in-review
priority: 3
created: 2026-10-02
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-46-the-gmail-pacer-stays-inside-googles-window-d46e45-1790978591983","branch":"adw/sabado-46-the-gmail-pacer-stays-inside-googles-window-d46e45","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-46-the-gmail-pacer-stays-inside-googles-window-d46e45-1790978591983/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1355","provider":"claude","model":"claude-sonnet-5-5","rebased":"875fe793318e86fc9e0cd1cb7edbeea6b23fe5d7","ciRounds":1}]
---
# chore(gmail): the pacer stays inside Google's window, so a read stops buying 403 cliffs

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3, Wave B
> row 42. **Config only — the enforcing mechanism already shipped** (PR #809,
> "the fetch stops losing a fifth of the mailbox — pacing, backoff, and batched
> headlines").

## What happens today

The pacer runs at the edge of Google's quota window rather than inside it. The
forensics bounded the cost at **8–17 minutes of 403 cliffs** per real-mailbox
read — time spent backing off from a limit the pacer could have stayed under.

Two numbers, both configuration, both already honoured by shipped code:

| | today | proposed |
|---|---|---|
| quota units | 5000 | **4500** |
| headline batch | 25 | **10** |

## Requirements

- [ ] **R1** The two values move to the proposed figures, through whatever
      configuration mechanism PR #809 already established. **No new mechanism,
      no new flag, no code path added.**
- [ ] **R2** Find and change **every** place each value is stated. If a value is
      written twice, say so in the PR body — a second copy is the drift this
      repo's own contracts exist to prevent, and may deserve its own ticket.
- [ ] **R3** If either value is environment-driven, the shipped default is the
      new one and `deploy/secrets.env.example` documents it beside its siblings.
      An unset value must not fall back to the old figure.
- [ ] **R4** No behaviour change beyond pace. Batching shape, backoff policy,
      retry semantics and what is fetched are untouched.

## Files

The pacer's config module · `deploy/secrets.env.example` (only if R3 applies) ·
`backend/tests/`. Nothing else.

## Verify

- [ ] A test asserts the effective quota-unit budget is 4500 and the headline
      batch is 10.
- [ ] `git grep -n '5000\|batch.*25' ` across the Gmail fetch path returns
      nothing stale — **quote the grep in the PR body**.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Measuring the saving (no mailbox read is dispatched by this ticket). Changing
the backoff. Any lane-count or concurrency tuning. The OCR bucket's own 60
pages/min per-key limit, which no lane tuning touches.
