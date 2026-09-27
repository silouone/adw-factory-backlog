---
id: sabado-01-two-test-gates-and-a-claimed-redis-slot
type: chore
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 150, turns: 450, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-01-two-test-gates-and-a-claimed-redis-slot-1789842707826","branch":"adw/sabado-01-two-test-gates-and-a-claimed-redis-slot","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-01-two-test-gates-and-a-claimed-redis-slot-1789842707826/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1217","provider":"claude","model":"sonnet"}]
---
# fix(scripts): just test can run one suite at a time, and a Redis slot is claimed, not guessed

> **Why, from the first sabado run.** `sabado-00` blocked after three repair
> rounds on two pytest failures the repair agent never saw: `just test` prints
> pytest, then ~3 400 vitest lines, and the factory hands repair the last
> 2 000 characters of the gate — a green vitest tail. With one gate per
> runner each failure lands in its own tail. And `resolve_test_slot` picks a
> Redis db as a pure hash over 14 databases: seven concurrent wave-1 runs have
> better-than-even odds that two share one, and `conftest.py:111` flushes it.
> `review: false`: every Verify bullet is a command or a red test.

## Requirements

- [ ] **R1** `scripts/sabado test [backend|frontend]`: one argument selects a
      suite; no argument keeps today's behaviour (both, `rc` propagated).
      `just test *args` passes it through.
- [ ] **R2** In `cmd_test`, when a slot is set, the Redis db is **claimed**:
      starting from the hash candidate, walk 1–14 and take the first db where
      `SET sabado:test-slot:<db> <slot> NX EX 7200` succeeds (or the key
      already holds this slot); none free → loud error naming all 14 owners.
      The claim is released (`DEL`, only if still owned by this slot) when
      `cmd_test` exits, on any path. `redis-cli` through
      `docker compose exec -T redis`, the same idiom `ensure_test_db` uses.
- [ ] **R3** `test-env` stays read-only: it prints the hash candidate and
      says so in a comment line; the claimed db is printed by `cmd_test`
      itself before pytest starts.
- [ ] **R4** `test-prune` also deletes stale `sabado:test-slot:*` keys.

## Files

`scripts/sabado` · `justfile` (the `test` recipe) ·
`backend/tests/test_scripts_test_loop.py`. Nothing else.

## Verify

- [ ] Red test (`@needs_dev_machine`): with `sabado:test-slot:<candidate>`
      pre-set to another slot's name, `cmd_test`'s printed claim is the next
      free db; after the run the key for the claimed db is gone and the
      foreign key is untouched. RED today.
- [ ] Red test: `scripts/sabado test nonsense` exits non-zero naming the
      accepted values. RED today.
- [ ] `SABADO_TEST_SLOT=<slot> scripts/sabado test backend` runs pytest only;
      `… test frontend` runs vitest only; both exit codes honest.
- [ ] Target gates green. **Operator, after merge:** replace the `test` gate
      in `targets/sabado.json` with `test-backend` (`… just test backend`)
      and `test-frontend` (`… just test frontend`), in that order.
