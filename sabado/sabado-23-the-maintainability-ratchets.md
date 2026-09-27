---
id: sabado-23-the-maintainability-ratchets
type: feat
status: done
priority: 3
created: 2026-09-19
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side, sabado-14-nothing-red-or-drifted-lives-on-main]
attempts: [{"runId":"sabado-23-the-maintainability-ratchets-1789905465452","branch":"adw/sabado-23-the-maintainability-ratchets","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-23-the-maintainability-ratchets-1789905465452/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1229"}]
---
# ci: god files cannot grow, a third copy cannot land, and the backend gets its own ds:check

> **Audit:** guardrail plan §5 #8 and the non-security half of §5 #1 —
> Lane I. Closes the ratchet halves of **P1-17** and **P1-22** and the
> grep rules `sabado-11` and `sabado-21` asked for. Depends on `sabado-14`
> because it adds two lines to `ci.yml`.

## What happens today

- The front has a real ratchet (`frontend/scripts/ds-debt-check.mjs`,
  baseline `frontend/scripts/ds-debt-baseline.json`, 11 counters that may
  only drop). The backend has none: `BACKEND.md` states rules that nothing
  checks; `ci.yml:147` says the mypy override list "CAN ONLY SHRINK" and
  nothing counts it (65 → 65 in 42 days).
- Files over 1 000 lines at HEAD (`wc -l`): `enrich.py` 4 343 (317 on
  07-17 — ×13.7 in two months), `SharedDocsPage.tsx` 1 711,
  `RecordDocumentsTable.tsx` 1 632, `household.py` 1 497, `rate_limit.py`
  1 226, `facts.py` 1 208, `ContractRecord.tsx` 1 107, `documents.py`
  1 085, `proposals.py` 1 082, `service.py` 1 076, `CalendarPage.tsx` 1 042
  (generated contracts and `schema.d.ts` excluded). Nothing stops any of
  them growing.
- `SharedDocsPage.tsx` and `RecordDocumentsTable.tsx` share 123 identical
  code lines; `DesignSystem.mdx:39-41` admits "nothing watches for
  duplication".

## Requirements

- [ ] **R1 — size ratchet.** `scripts/size-ratchet.py` + `size-baseline.json`
      at the repo root: every tracked source file over 1 000 lines
      (excluding `frontend/src/contracts/*`, `schema.d.ts`, tests, and
      migrations) may not grow past its baseline; `--update-baseline`
      tightens. Runs on both trees.
- [ ] **R2 — the front counters.** `ds-debt-check.mjs` gains
      `clientImportsOutsideHooks` (`api/client` imported outside
      `hooks/`/ports), `pagesImportedFromOutside` (`pages/**` imported from
      `components/`, `hooks/`, `lib/`), `stores` (count of zustand stores;
      4 today), and `duplicatedBlocks` from `jscpd --min-tokens 70 src/pages
      src/components` (jscpd as a devDependency); baseline updated with
      today's values.
- [ ] **R3 — the backend's ds:check.** `backend/scripts/api-debt-check.py`
      + `api-debt-baseline.json`, counters that may only drop: routes
      without `response_model`/`response_class` excluding 204s; router
      files over 600 lines; routers without a test file; `[mypy-app.`
      sections in `mypy.ini`; JSON columns; `create_refresh_token(` call
      sites outside `_issue_tokens`; `User.email ==` / `invited_email ==`
      matches outside `auth/` without `email_verified` on the same path.
      Prints each counter and its baseline; `--update-baseline` tightens.
- [ ] **R4 — wired where checks run.** `just check` runs the three scripts;
      `ci.yml` gains one line in `front` and one in `back`.
- [ ] **R5 — prose becomes a pointer.** The `BACKEND.md` sentences that
      state a now-counted rule point at the script instead of restating it.

## Files

`scripts/size-ratchet.py` (new) · `size-baseline.json` (new) ·
`frontend/scripts/ds-debt-check.mjs` · `frontend/scripts/ds-debt-baseline.json`
· `frontend/package.json` (jscpd devDependency) · `frontend/package-lock.json`
· `backend/scripts/api-debt-check.py` (new) · `backend/api-debt-baseline.json`
(new) · `justfile` (`check` recipe; `sabado-00` added `test-prune` elsewhere)
· `.github/workflows/ci.yml` (two lines) ·
`.claude/skills/sabado-project/BACKEND.md` (pointers only) · one test per
script.

## Verify

- [ ] Red test: `backend/tests/test_api_debt_check.py` runs the script on
      a fixture tree with one route lacking `response_model` above the
      baseline → non-zero naming the counter. RED today (no script).
- [ ] Red test: `scripts/size-ratchet.py` on a temp copy where one
      baseline file has one extra line → non-zero naming the file. RED.
- [ ] `node scripts/ds-debt-check.mjs` green with the new counters at
      baseline; the same with one extra `api/client` import in a page →
      red naming `clientImportsOutsideHooks`.
- [ ] `just check` green on HEAD.
- [ ] The counters that `sabado-11` / `sabado-21` zeroed read 0 in the
      baselines if those tickets merged first; their pre-merge values
      otherwise — never a hand-typed number.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.

## Out of scope

The `enrich.py` split (Lane K), the `useDocumentSelection` hook
(`sabado-24`), the route-sweep 401 test (§5 #1(a), security pass),
`untested-routes` (§5 #9), coverage thresholds.
