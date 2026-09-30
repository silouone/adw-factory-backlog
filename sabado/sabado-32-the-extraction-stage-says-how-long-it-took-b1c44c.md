---
id: sabado-32-the-extraction-stage-says-how-long-it-took-b1c44c
type: bug
status: in-review
priority: 1
created: 2026-09-30
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-32-the-extraction-stage-says-how-long-it-took-b1c44c-1790770590207","branch":"adw/sabado-32-the-extraction-stage-says-how-long-it-took-b1c44c","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-32-the-extraction-stage-says-how-long-it-took-b1c44c-1790770590207/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1325","provider":"claude","model":"claude-sonnet-5-5"}]
---
# fix(extraction): the extraction stage is timed, so the 15-minute cap can be reasoned about at all

> **Finding:** `ai_docs/2026-09-30-sabado-ingestion-delta.md` §2. **Fire this
> first, alone.** It is the measurement every other ingestion ticket is judged
> by; landing an optimisation before it means the optimisation cannot be shown
> to have worked.
>
> **Shares two files with other tickets.** `cost.py` with `sabado-33` (which
> owns the *pricing* region, ~`:660-690`); `household_read.py` with
> `sabado-35` (which owns the *founding-lane* region, `:483-484`). This
> ticket's regions are the `STAGE_*` constant block (~`:60-75`) and the
> `perform_read` call sites listed under Files.

## What happens today

`cost.timing(stage)` (`cost.py:526`) folds seconds into `meter.stage_seconds`
(`cost.py:125`, `:520`), exported by `cost.snapshot()` under `"stage_seconds"`
(`cost.py:702`) and written to `row.cost` (`household_read.py:613`). The
mechanism is real, persisted, and already used by five stages:
`STAGE_SECTOR_SEARCH`, `STAGE_HEADLINE_FETCH`, `STAGE_TRIAGE`,
`STAGE_BODY_FETCH`, `STAGE_ATTACHMENT_ANALYSIS`.

**`STAGE_EXTRACTION` is defined at `cost.py:71` and has zero call sites.**
Verified on `origin/main`:

```
$ git grep -n "STAGE_EXTRACTION" origin/main -- backend/app
origin/main:backend/app/extraction/cost.py:71:STAGE_EXTRACTION = "extraction"
```

That is the largest stage in the pipeline. On Erwann's 2 h 27 run, extraction
+ founding + state + routing together were **≈4 300 s — 49 % of wall clock —
and none of it timed** (`tickets/findings/2026-09-21-ingestion-under-15-min.md`
§5). The forensics named this gap on 2026-09-21 and nothing has been added
since: `git log -S"cost.timing"` → `2290b5cc`, 2026-09-08.

Four other spans on the critical path are also untimed: `_founding_lane`
(`household_read.py:483`), `gate.write` (`:521`), `_state_lane` (`:551`), and
routing (`route_inputs`, `:835`).

## Requirements

- [ ] **R1** `household_facts_from_messages` (`household_read.py:498`) runs
      inside `with cost.timing(cost.STAGE_EXTRACTION):`. This one line is the
      ticket's reason to exist.
- [ ] **R2** Three new constants beside the existing block in `cost.py`:
      `STAGE_FOUNDING`, `STAGE_LEDGER_WRITE`, `STAGE_ROUTING` — same naming
      and same string-value convention as their five siblings.
- [ ] **R3** Those three wrap `_founding_lane` (`:483`), `gate.write` (`:521`)
      and `route_inputs` (`:835`) respectively.
- [ ] **R4** No behaviour change, no new dependency, and `stage_seconds` keeps
      its existing shape: a flat `dict[str, float]` accumulated per run. Do
      **not** add per-document or per-batch attribution here — that needs a
      schema change and is out of scope.
- [ ] **R5** A stage that raises still records its elapsed time (`timing` is a
      context manager; confirm it does not swallow, and do not make it).

## Files

`backend/app/extraction/cost.py` (the `STAGE_*` constant block only) ·
`backend/app/extraction/household_read.py` (`perform_read` at `:483`, `:498`,
`:521`, and `route_inputs` at `:835`) · `backend/tests/test_extraction.py` or
a new `backend/tests/test_stage_timing.py`. Nothing else.

## Verify

- [ ] **The red test, first.** A test that runs a read and asserts
      `"extraction" in snapshot()["stage_seconds"]` **fails on the current
      tree** — that failure is the ticket. Then the same for the other three
      stage keys.
- [ ] After: all four keys present, each a positive float.
- [ ] The five pre-existing stage keys are still present and unchanged in
      name. Print the whole `stage_seconds` dict in the PR body.
- [ ] `git diff --stat` touches only the files listed above.
- [ ] Target gates: `mypy app/` · `lint-imports` · `api-debt-check` ·
      `alembic heads` · `alembic-branch-check` · `test` — all green.

## Out of scope

Per-document or per-batch timing rows (needs a migration). Enforcing a
15-minute budget (needs commit-per-batch first). Any optimisation of the
stages being timed — `sabado-35` owns the founding lane.
