---
id: sabado-29-every-pr-walks-a-real-browser-through-a-real-backend-a98ee9
type: feat
status: in-progress
priority: 1
created: 2026-09-28
caps: {minutes: 240, turns: 700, stallMinutes: 25}
depends: [sabado-27-the-built-front-can-talk-to-a-real-backend-acabfa, sabado-28-a-throwaway-account-is-born-verified-and-deleted-ca5fba]
attempts: []
---
# test(ci): every PR walks a real browser through a real backend, so a contract drift goes red before it merges

> **Why the job and the first journeys are one ticket.** A CI job with nothing
> to run is not verifiable, and a journey with no job to run it in is not
> either. This ticket is the job plus the two journeys that prove it works.
>
> **Why CI and not the local loop first (operator, 2026-09-28).** This suite
> exists to protect a second contributor's iteration *before* the fact, and to
> let a refactor land safely. Protection that only runs on the operator's Mac
> protects neither. The factory profits from the same suite through its
> `test-e2e-boot` gate at the same time.

## What happens today

`ci.yml` has three jobs, and the gap is structural:

- **`front`** (`needs: changes`, `if: front == 'true'`) runs `ds:check`,
  typecheck, lint, format, `npm run build`, `npx playwright install --with-deps
  chromium`, `npm run test:e2e`, vitest, build-storybook. Its e2e is the
  **stubbed bundle** suite — no backend.
- **`back`** (`if: back == 'true'`) already runs **`postgres:16` + `redis:7` as
  services** and `pytest -q`. The real stack in CI is therefore not a new
  capability; nothing has ever pointed a browser at it.
- **`main`** (on push) runs `main-check.sh` only — four static checks. **Nothing
  runs a browser against the merged tree**, which is exactly the 2026-09-20
  incident class: two independently-green PRs, a broken combination, a blank
  page on every route for half an hour with 3 450 unit tests green.

And the path gate cuts the other way too: **a backend-only PR that breaks a
screen gets no frontend check at all**, because `front` is skipped.

## Requirements

- [ ] **R1 — a new `e2e` job, triggered by either side.**
      `if: needs.changes.outputs.front == 'true' || needs.changes.outputs.back == 'true'`.
      That single line is half the reason this job exists: it closes the
      backend-only-PR hole above. Runs in parallel with `front` and `back`, not
      after them.
- [ ] **R2 — the stack it stands up.** `postgres:16` + `redis:7` as
      `services:`, copied from the `back` job's block including the health
      options. Then, in order: `pip install -r backend/requirements.txt`
      (**not** `-dev` — mypy and import-linter never run here),
      `alembic upgrade head`, `uvicorn app.main:app --port 8010` in the
      background, **poll `/health` until it answers** (never a fixed sleep),
      `npm ci`, `npm run build`, `npx playwright install --with-deps chromium`,
      the suite. Port 8010 matches `vite.config.js:20-21`'s local convention.
- [ ] **R3 — the environment it needs, stated not assumed.** `conftest.py`
      supplies pytest's defaults and **none of them reach a uvicorn this job
      launches itself**. The job must export: `DATABASE_URL` and `REDIS_URL`
      pointing at the services, `SECRET_KEY` (**one value**, shared by the
      mint route and `verify_email_code` — they HMAC with it), `STORAGE_PATH`,
      `ENVIRONMENT` set to anything but `production`, and **`ADMIN_TOKEN` to a
      throwaway literal** — `require_admin` answers **503 when it is empty**
      (`admin.py:54-58`), so without it every account-fixture call fails. These
      are plain `env:` values against ephemeral service containers, **not
      repository secrets**: nothing here exists after the job ends.
- [ ] **R4 — the switch from `sabado-27`.** The job sets
      `E2E_API_TARGET=http://localhost:8010` and `E2E_BASE_URL`, which is what
      includes the `journeys` project. The `front` job is **left alone** — its
      `npm run test:e2e` keeps running `bundle` only, and keeps its ~2 min.
- [ ] **R5 — journey 1: register → verify.** `e2e/journeys/01-register.spec.ts`.
      `/register` with a password → onboarding; the code minted by `sabado-28`
      R1 through the real `POST /auth/verify-email` → `GET /auth/me` reports
      `email_verified: true`. No `page.route` anywhere.
- [ ] **R6 — journey 2: the session holds.** `e2e/journeys/02-session.spec.ts`.
      Login → `/dashboard` renders → reload (the `sabado_refresh` cookie
      rotates, `SESSION.md`) → logout → `/dashboard` redirects to `/welcome`,
      **and the old refresh cookie no longer works** (the blocklist is
      honoured). This is the P1-2 logged-out bug `#1230` fixed; it is the
      journey that proves the suite catches a real regression.
- [ ] **R7 — a failure names the journey and keeps the trace.**
      `trace: 'retain-on-failure'` on the `journeys` project, uploaded with
      `actions/upload-artifact`; the step summary lists each journey with
      pass/fail (`sabado-26` R5).
- [ ] **R8 — a stated budget.** Target **under 8 minutes** wall-clock for the
      job. Record the first green run's actual duration in the PR body. If it
      exceeds 8 min, say so in the PR rather than trimming a journey — the
      operator decides whether to pay it.

## Files

`.github/workflows/ci.yml` (the `e2e` job, R1–R4, R7) ·
`frontend/e2e/journeys/01-register.spec.ts` (new, R5) ·
`frontend/e2e/journeys/02-session.spec.ts` (new, R6) ·
`frontend/playwright.config.ts` (R7, trace on the `journeys` project only).

## Verify

- [ ] **Red first:** journey 2 against a tree at `8169a600` (main before
      `#1230`) fails on the refresh-rotation case; green on current `main`.
      This is the ticket's central claim — a real regression, caught.
- [ ] Both journeys green in the `e2e` job, with **zero `page.route` calls** in
      `e2e/journeys/` (`grep -rn "page.route" frontend/e2e/journeys/`).
- [ ] A **backend-only** PR (touching `backend/` and nothing under `frontend/`)
      runs the `e2e` job. Proven by an actual PR, not by reading the `if:`.
- [ ] The `front` job's duration is unchanged, and its `npm run test:e2e` still
      reports only the `bundle` project's test count.
- [ ] A deliberately broken selector shows a red step, names the journey, and
      attaches a downloadable trace.
- [ ] `GET /admin/accounts` no longer lists the throwaway account after the job,
      **including on a run where a journey failed**.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `test-e2e-boot` · `just test` — green.

## Out of scope

Journeys 3–8 (`sabado-30`) · the calendar/biens/foyer journeys (`sabado-31`) ·
running any of this against a deployed slot (`sabado-26`) · adding e2e to the
post-merge `main` job — the merged-tree hole is real and named above, but it is
a separate decision about minutes on a private repo's quota.
