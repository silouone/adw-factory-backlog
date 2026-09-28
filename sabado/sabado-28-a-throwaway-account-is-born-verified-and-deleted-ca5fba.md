---
id: sabado-28-a-throwaway-account-is-born-verified-and-deleted-ca5fba
type: feat
status: in-progress
priority: 1
created: 2026-09-28
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-28-a-throwaway-account-is-born-verified-and-deleted-ca5fba-1790594011185","branch":"adw/sabado-28-a-throwaway-account-is-born-verified-and-deleted-ca5fba","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-28-a-throwaway-account-is-born-verified-and-deleted-ca5fba-1790594011185/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1312","provider":"claude","model":"sonnet"}]
---
# feat(admin): a throwaway account can be minted a verification code and deleted, so an e2e suite owns its own user

> **This ticket carries an amendment to `sabado-26` R2.** The full reasoning is
> in adw-factory `ai_docs/2026-09-28-sabado-26-r2-amendment-email-code.md`. The
> short version: R2 says "the code lives in Redis" and asks for
> `GET /admin/accounts/{id}/email-code`. **Redis holds
> `hmac_sha256(SECRET_KEY, "<user_id>:<code>")`, not the code**
> (`backend/app/auth/redis_store.py:75-89`) — the transform is one-way, so that
> GET has nothing to return and cannot be built. The mailer logs no code either:
> `backend/app/core/mailer.py:42` logs exactly `"SMTP not configured: e-mail
> ignored (clean degradation)."`. The route below therefore **mints** a code
> rather than reading one. The 404-in-production guard, which is the security
> half of decision D1, is unchanged.

## What happens today

- Email verification: `POST /auth/verify-email` (`auth/router.py:571`) →
  `verify_email_code(user_id, code)` → HMAC compare, 5 attempts then the code is
  burned (`redis_store.py:91-110`).
- `require_admin` (`admin.py:48`) reads `X-Admin-Token` against
  `settings.ADMIN_TOKEN` and answers **503 when that token is empty** — admin is
  disabled by default, never open by accident.
- `DELETE /admin/accounts/{user_id}` already exists (`admin.py:269`).
- The persona QA (`scripts/qa-*`) is a human gesture on a **shared** account
  that aliases a real Gmail, which is why guardrail 6 forbids destructive
  journeys on it. An automated suite cannot use it.

## Requirements

- [ ] **R1 — mint, do not read.** `POST /admin/accounts/{user_id}/email-code`
      behind `require_admin`, following the `admin.py:88-269` pattern. It mints a
      fresh code with the router's own `_new_email_code()`, stores it via
      `store_email_code`, and returns the plaintext. Reuse `_new_email_code` —
      do not re-implement the alphabet.
- [ ] **R2 — 404 where it must not exist.** When `ENVIRONMENT == "production"`
      the route answers **404**, not 403: in production it does not exist.
      Checked before `require_admin`'s work, so a leaked admin token learns
      nothing from the status code.
- [ ] **R3 — a silent store is a loud failure.** `store_email_code` swallows
      `redis.RedisError` (`redis_store.py:87-88`). The route must confirm the
      `ev:{user_id}` key landed and return **503** naming the failure if it did
      not. Without this, a Redis blip returns a plaintext code whose hash was
      never stored, and journey 1 fails at `verify-email` with nothing to read.
- [ ] **R4 — the account fixture.** `frontend/e2e/fixtures/account.ts` exports a
      Playwright fixture that:
      1. registers `qa-e2e+<runId>@sabado-e2e.invalid` with a password, where
         `<runId>` makes concurrent runs disjoint;
      2. mints a code via R1 and verifies the address through the real
         `POST /auth/verify-email`;
      3. exposes `{ email, password, userId }` to the spec;
      4. deletes the account via `DELETE /admin/accounts/{userId}` in a
         **`test.afterAll` that runs even when a journey threw** (`sabado-26`
         R6).
      It reads `E2E_ADMIN_TOKEN` and **throws a named error if it is unset** —
      a 503 from `require_admin` is otherwise indistinguishable from a bug.
- [ ] **R5 — it refuses to run against production.** The fixture throws before
      registering anything if `E2E_BASE_URL` resolves to `app.sabado.io`
      (`sabado-26` R6). One vitest case freezes that refusal.
- [ ] **R6 — the shared QA account is never touched.** The fixture has no code
      path that reads `scripts/qa-*` or any persona file. Guardrail 6 therefore
      does not apply to this suite.

## Files

`backend/app/api/admin.py` (R1–R3) · `backend/tests/test_admin.py` (R1–R3) ·
`frontend/e2e/fixtures/account.ts` (new, R4–R6) ·
`frontend/src/**/*.test.ts` (one case for R5's refusal).

## Verify

- [ ] **Red test first (Art. I):** `POST /admin/accounts/{id}/email-code` with
      `ENVIRONMENT=production` → 404; without it → a code that
      `POST /auth/verify-email` accepts; with Redis unreachable → 503 naming the
      store failure. All three RED today (route absent).
- [ ] The minted code round-trips: mint → `verify-email` → `GET /auth/me` shows
      `email_verified: true`.
- [ ] A second mint **invalidates the first code** (`setex` overwrites
      `ev:{user_id}`), asserted explicitly — this is the property that makes the
      route safe to expose to an admin.
- [ ] `ADMIN_TOKEN` empty → 503 from `require_admin`, and the fixture's own
      error names `E2E_ADMIN_TOKEN` rather than surfacing a bare 503.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `test-e2e-boot` · `just test` — green.

## Out of scope

Any journey spec, any CI wiring — `sabado-29`. The fixture is written here and
first *used* there.
