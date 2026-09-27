---
id: sabado-17-one-request-cannot-freeze-the-api
type: bug
status: done
priority: 2
created: 2026-09-19
review: false
caps: {minutes: 180, turns: 500, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-17-one-request-cannot-freeze-the-api-1789850667705","branch":"adw/sabado-17-one-request-cannot-freeze-the-api","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-17-one-request-cannot-freeze-the-api-1789850667705/workspace","outcome":"blocked"},{"runId":"sabado-17-one-request-cannot-freeze-the-api-1789859147803","branch":"adw/sabado-17-one-request-cannot-freeze-the-api","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-17-one-request-cannot-freeze-the-api-1789859147803/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1223"}]
---
# fix(api): no handler blocks the event loop, and the children list stops issuing a query per member

> **Audit:** **P1-20** (axis L) + the `/children` N+1 (L P2) — Lane E.
> `review: false`: hard-gated.

## What happens today

`backend/app/api/ai.py:91` `async def analyze`; `:117-121` `_prepare_pdf` /
`_prepare_image` are synchronous CPU work; `:134` `provider.analyze(...)`
is called **without `await`** — `ai/provider.py:162,405` are plain `def`
and the Mistral client's timeout is 300 s (`provider.py:314`).
`backend/Dockerfile:38` runs one uvicorn process, no `--workers`. While one
document is analysed, **no other request is dispatched** — not the sync
routes waiting for the loop to hand them to the threadpool, not
`GET /health`, which `ingestion-alarm.sh:14` reads every 5 minutes and
reports down after two misses.

Same pattern, smaller cost: `documents.py:139-155` and
`shared_documents.py:601-611` write up to 25 MB with a blocking
`open().write()` inside `async def`.

`children.py:290` `list_children` loads the members then lazy-loads two
relations per member (a documented N+1); `assets.py:238` already does the
right thing with `selectinload`.

No Python linter exists in the repo; `flake8-async`'s ASYNC210/ASYNC230
would have flagged all three sites.

## Requirements

- [ ] **R1** `analyze` becomes `def` (the only `await` was `file.read()`;
      `file.file.read()` replaces it) and runs in the threadpool like `chat`
      does. Same for the two upload handlers (or their write goes through
      `await anyio.to_thread.run_sync`).
- [ ] **R2** `backend/ruff.toml` selects **only** the `ASYNC` rule family;
      `ruff` is pinned in `backend/requirements-dev.txt`. No other ruff rule
      is adopted by this ticket.
- [ ] **R3** The rule is enforced where the gates already run: a pytest test
      executes `ruff check --select ASYNC app/` and fails on any finding —
      so the factory `test` gate and the CI `back` job both carry it
      **without editing `ci.yml`** (`sabado-14` owns that file; Lane I may
      promote this to its own step later).
- [ ] **R4** `list_children` uses `selectinload(Child.documents)` and
      `selectinload(Child.reminders)` (the `assets.py:238` idiom).
- [ ] **R5** A reusable query-count fixture in `conftest.py`
      (`before_cursor_execute` listener on the engine) — the one thing this
      ticket adds to the shared conftest.

## Files

`backend/app/api/ai.py` · `backend/app/api/documents.py` ·
`backend/app/api/shared_documents.py` · `backend/app/api/children.py` ·
`backend/ruff.toml` (new) · `backend/requirements-dev.txt` ·
`backend/tests/conftest.py` (the fixture only) ·
`backend/tests/test_children.py` · `backend/tests/test_lint_async.py` (new).
The agent installs `ruff` into the venv itself (`uv pip install ruff==<pin>
--python "$SABADO_VENV/bin/python"`) — bootstrap ran before this branch
existed.

## Verify

- [ ] Red test: `test_lint_async.py::test_no_blocking_call_inside_an_async_handler`
      — subprocess `ruff check --select ASYNC app/`, assert exit 0. RED
      today on the three sites.
- [ ] Red test: `test_children.py::test_listing_five_members_costs_at_most_four_queries`
      — five members with documents and reminders, `GET /children`, the
      fixture counts ≤ 4 statements. RED today.
- [ ] `inspect.iscoroutinefunction(analyze)` is false (asserted in
      `test_lint_async.py` as a second, cheap guard).
- [ ] Every existing test in `test_ai.py`, `test_documents.py`,
      `test_shared_documents.py`, `test_children.py` stays green.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

`--workers`; the other 27 unpaginated lists (L P2 ratchet); any ruff rule
beyond `ASYNC`; `ci.yml`.
