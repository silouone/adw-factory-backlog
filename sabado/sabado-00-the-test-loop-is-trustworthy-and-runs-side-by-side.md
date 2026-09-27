---
id: sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side
type: feat
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side-1789822243911","branch":"adw/sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side-1789822243911/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side-1789823904374","branch":"adw/sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side-1789823904374/workspace","outcome":"in-review (salvaged by hand)","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1216"}]
---
# fix(scripts): just test runs the backend, propagates its exit code, and runs N times side by side

> **Audit:** P1-21 (axes H, Q, C) — Lane A of the LANES plan, plus the one
> change that lets the factory run several sabado tickets at once.
> **Wave 0 — runs alone.** Every other ticket `depends:` on this one.

## Why this is first

Four defects in the local loop, all observed this week:

1. `scripts/bootstrap:60-64` strips `pillow-heif` on Darwin ("its build fails
   there") while `backend/requirements.txt:46-47` records that 1.5.0 installs
   on macOS arm64 (verified 2026-08-08) and the wheel installs in seconds.
   `backend/tests/test_upload_parse_path.py:24` imports it unconditionally,
   on purpose (an `importorskip` "would turn an incomplete environment into a
   green suite"). Result on a fresh Mac: `Interrupted: 1 error during
   collection`, **0 backend tests run**.
2. `scripts/sabado:209-218` `cmd_test` runs pytest then vitest with
   `set -uo pipefail` (no `-e`), no `||`, no `rc=`. **`just test` exits with
   vitest's code whatever pytest did** — including the collection error
   above. This is the factory's `test` gate: a red backend is invisible to
   it today.
3. `cmd_test` calls `load_env`, which exports `REDIS_URL=redis://localhost:6379/0`
   (`scripts/sabado:89`) — the **dev** Redis. `conftest.py:18` only
   `setdefault`s `/15`, so the export wins, and `conftest.py:111` runs
   `get_redis().flushdb()`. Every `just test` wipes dev's Redis (session
   blocklist, rate limits, OAuth states).
4. No `.nvmrc`, no `engines`. CI pins Node 22 (`ci.yml`); the factory host
   and every new Mac run Node 25, where a native `localStorage` shadows
   jsdom's and 60 tests fail with `localStorage.clear is not a function`
   (`useScanStage.test.tsx`, `OnboardingV2.test.tsx`). Green with
   `NODE_OPTIONS=--no-experimental-webstorage`.

And the one that blocks parallel runs: `scripts/sabado:214` exports a single
fixed `DATABASE_URL=…/sabado_test`, and `conftest.py:45-46` does
`drop_all` + `create_all` on it. Two checkouts running `just test` at the
same time drop each other's schema mid-suite — a **silent false gate
verdict**, not an error.

## Requirements

- [ ] **R1 — pillow-heif installs everywhere.** Delete the Darwin branch in
      `scripts/bootstrap` (lines 60-64) and the header note (lines 8-10); the
      `else` path runs unconditionally. Bootstrap installs
      `backend/requirements-dev.txt` (which pulls `requirements.txt` in and
      adds mypy + import-linter) instead of `requirements.txt`.
- [ ] **R2 — `just test` tells the truth.** `cmd_test`: `rc=0; ( … pytest -q )
      || rc=1; ( … vitest run ) || rc=1; exit $rc`. Both suites still run
      when the first fails.
- [ ] **R3 — the test loop never touches dev state.** `cmd_test` exports its
      own `REDIS_URL` (db 15 by default, see R5) and `STORAGE_PATH`
      (`/tmp/sabado_test_storage` by default) explicitly, after `load_env`,
      the way it already exports `DATABASE_URL`.
- [ ] **R4 — Node is pinned, advisorily.** `frontend/.nvmrc` = `22`;
      `frontend/package.json` `"engines": {"node": "22.x"}`. **No
      `engine-strict`** — the factory host runs Node 25 today and `npm ci`
      must keep working there. The vitest invocation `cmd_test` uses (and the
      `test` npm script) sets `NODE_OPTIONS=--no-experimental-webstorage`,
      a no-op on 22, the fix on 25. `ci.yml` is not touched.
- [ ] **R5 — test slots.** `cmd_test` honours `SABADO_TEST_SLOT` (optional,
      `^[a-f0-9]{4,12}$`, anything else is a loud error). When set:
      database `sabado_test_<slot>` (created on demand through
      `docker compose … exec -T postgres createdb`, the idiom bootstrap's
      `create_db` already uses), Redis db `((16#<first two hex chars> % 14) + 1)`
      (1–14; dev keeps 0, the default slot keeps 15), storage
      `/tmp/sabado_test_storage_<slot>`. Unset → today's names, unchanged.
- [ ] **R6 — slots can be reclaimed.** `scripts/sabado test-prune` drops every
      `sabado_test_*` database and removes every
      `/tmp/sabado_test_storage_*` directory; `just test-prune` delegates to
      it (the justfile rule: it delegates, logic lives in `scripts/`).
- [ ] **R7 — bootstrap's acceptance proves the loop.** `scripts/agent-env.sh`
      additionally runs `"$VENV/bin/python" -c 'import pillow_heif'` and
      `pytest --collect-only -q >/dev/null`, and **warns** (never fails) when
      `node --version`'s major differs from the one `ci.yml` pins — read
      from the file, not repeated.
- [ ] **R9 — a worktree reuses the host's containers.** Every
      `docker compose` call in `scripts/bootstrap` and `scripts/sabado`
      pins the project name (`-p sabado`, or a `COMPOSE_PROJECT_NAME`
      default set at the top of both scripts). Compose derives the project
      from the directory name, so a checkout named `workspace` (a factory
      worktree) tried to create `workspace-postgres-1` on :5432 and died
      with "port is already allocated" — the first sabado run blocked there
      (attempt 1 below). Until this lands the target's `setup` exports the
      variable itself.
- [ ] **R8 — stale sentences go.** `ci.yml:132-137`'s inverse remark about
      the wheel is left alone (Lane B owns `ci.yml`); the bootstrap header
      and the `scripts/sabado` comment at `:213` say what is now true.

## Files

`scripts/bootstrap` · `scripts/sabado` · `scripts/agent-env.sh` · `justfile`
(one recipe) · `frontend/.nvmrc` (new) · `frontend/package.json` (`engines`
+ the `test` script) · one new test file per side (below). Nothing under
`backend/app/`, nothing under `.github/`.

## Verify

- [ ] Red test (backend): `backend/tests/test_scripts_test_loop.py` runs
      `scripts/sabado test-prune --dry-run` and a `SABADO_TEST_SLOT=zz`
      invocation through `subprocess` and asserts the loud error; and asserts
      that with `SABADO_TEST_SLOT=abcd1234` the script's resolved env (a
      `scripts/sabado test-env` subcommand that only prints the three
      exports) names `sabado_test_abcd1234`, redis db `(16#ab % 14)+1`, and
      the slot storage path. RED today (no subcommand, no slot).
- [ ] Red test (frontend): `frontend/scripts/engines.test.ts` asserts
      `.nvmrc` trimmed equals the major in `package.json` `engines.node` and
      equals the `node-version` in `.github/workflows/ci.yml`. RED today.
- [ ] `SABADO_TEST_SLOT=$(printf %s "$PWD" | shasum | cut -c1-8) just test`
      runs 2 111 + 3 428 tests green on this host (Node 25).
- [ ] Two checkouts, two slots, `just test` at the same time: both green.
- [ ] From a checkout whose directory is not named `sabado`, `scripts/bootstrap`
      reports the existing containers healthy and creates none
      (`docker ps -a | grep -c workspace` → 0).
- [ ] `just test` after `pytest -k nonexistent_xyz` is forced red (temporarily
      edit `cmd_test`, or run `( cd backend && pytest -k nonexistent_xyz )`
      through the same `rc=` path): exit code non-zero.
- [ ] `docker exec … redis-cli -n 0 dbsize` is unchanged by a `just test`.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

`ci.yml` (Lane B, `sabado-14`), the pre-push hook (same), `engine-strict`,
any change under `backend/app/` or `frontend/src/` beyond the test file.
