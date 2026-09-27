---
id: sabado-13-models-and-migrations-agree
type: bug
status: done
priority: 2
created: 2026-09-19
review: false
caps: {minutes: 240, turns: 700, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-13-models-and-migrations-agree-1789850667702","branch":"adw/sabado-13-models-and-migrations-agree","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-13-models-and-migrations-agree-1789850667702/workspace","outcome":"blocked"},{"runId":"sabado-13-models-and-migrations-agree-1789857941494","branch":"adw/sabado-13-models-and-migrations-agree","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-13-models-and-migrations-agree-1789857941494/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1225"}]
---
# fix(db): the models and the migration chain describe the same schema, and a test proves it

> **Audit:** **P1-18** (axes E, H) — the first half of Lane B, split out so it
> can run beside `sabado-14`. Re-verified by the synthesis: `alembic check`
> on the dev database migrated to `0067` → **38 operations** (13
> `remove_index`, 24 `modify_nullable`, 1 `modify_type`). `review: false`:
> every Verify bullet is a red test or a gate.

## What happens today

- 13 foreign-key indexes exist in the database but are not declared on the
  models. The next `alembic revision --autogenerate` — the textbook agent
  move — emits a migration that **drops** them. CI is green (`alembic heads`
  only), boot applies it.
- 24 timestamp columns are `NOT NULL` on the models and nullable in the
  database (they all carry `server_default=NOW()`); the same autogenerate
  emits 24 `SET NOT NULL`s nobody reviewed. One column differs in type.
- Tests build the schema with `create_all` (`backend/tests/conftest.py:44-46`),
  so the 71-revision chain runs for the first time at container boot, and
  the suite runs against a schema production does not have.
- `alembic/env.py` sets no `lock_timeout`; a migration waiting on a lock
  holds the deploy indefinitely.

## Requirements

- [ ] **R1** The 13 indexes are declared on the models (`Index(...)` /
      `index=True`), matching the database's names. **No migration** for
      them — they exist.
- [ ] **R2** One migration `ALTER … SET NOT NULL` for the 24 timestamps, plus
      the one type fix, generated from `alembic check`'s own list, reviewed
      line by line — nothing else in it. Its `down_revision` is the head of
      `origin/main` at branch time (`0067_event_participants` today);
      numeric prefix `0068`.
- [ ] **R3** `backend/alembic/env.py` runs `SET lock_timeout = '5s'` on the
      migration connection.
- [ ] **R4** A pytest test builds a scratch database from the **chain**
      (`alembic upgrade head` on `<DATABASE_URL database>_mig`, created on
      demand, dropped at the end) and asserts `alembic check` reports no
      operation. It runs in the normal suite, so the factory gate and the CI
      `back` job both enforce it without a workflow change.
- [ ] **R5** `alembic check` on the dev database after `just migrate` → clean.

## Files

`backend/app/models/*.py` (the 13 index declarations only) · one new file
in `backend/alembic/versions/` · `backend/alembic/env.py` ·
`backend/tests/test_migrations.py` (the new test; keep the existing ones).
**Not** `.github/`, not `conftest.py`, not any router.

## Verify

- [ ] Red test: `test_migrations.py::test_the_chain_produces_the_models_schema`
      — upgrade an empty scratch database through the chain, run
      `alembic check` in-process (`alembic.command.check`), assert it does
      not raise. RED today (38 operations).
- [ ] `alembic heads` → exactly one head, `0068_…`.
- [ ] `ls backend/alembic/versions | cut -c1-4 | sort | uniq -d` lists
      nothing new (the four pre-existing duplicates `0045 0046 0050 0064`
      are `sabado-14`'s baseline, not this ticket's).
- [ ] The migration downgrades and re-upgrades cleanly on the scratch db.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

The post-merge CI job, the `down_revision` PR check, the pre-push hook
(`sabado-14`); CHECK constraints and enums (E P2); renumbering the four
duplicate prefixes; `CONCURRENTLY` on index creation.
