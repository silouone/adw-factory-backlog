---
id: sabado-26-eight-journeys-a-user-can-take-are-walked-on-every-slot-deploy
type: feat
status: queued
priority: 1
created: 2026-09-20
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-25-the-built-front-boots-in-a-browser-before-it-ships]
attempts: []
---
# test(e2e): eight journeys a user can take are walked by Playwright on every slot deploy, and before every production promotion

> **Why now, and why P1.** The front is where most change lands now, from
> more than one contributor, and nothing walks a screen or a feature end to
> end before it ships: on 2026-09-20 staging served a blank page on every
> route for half an hour with 3 450 unit tests, the build and five deploy
> proofs all green. This suite is the safety net for features and screens,
> not a convenience. It is the audit's own draft (axis H, "Smallest
> Playwright smoke suite", `.claude/reports/audit-work/H.md`), ticketed as
> written, with its two open decisions taken below. Plumbing
> (`@playwright/test`, chromium in CI, `frontend/e2e/`) arrives with
> `sabado-25`.

## What happens today

109 page components, 38 with no test at all; the persona QA
(`scripts/qa-*`) is a human gesture on a shared account and may not perform
the destructive journeys. A login loop (P1-2), a vault field leaving the
browser in clear, a calendar feed that stops round-tripping, a Google SSO
redirect with the wrong `redirect_uri` — each has a backend test or a DOM
test; none is exercised through the real stack in a real browser.

## Requirements

- [ ] **R1 — its own account, always.** The suite registers
      `qa-e2e+<sha>@sabado-e2e.invalid` with a password, verifies it, and
      deletes it at the end through `DELETE /admin/accounts/{user_id}`
      (`admin.py:269`, `require_admin`). It never touches the shared QA
      account (guardrail 6 of the persona rules therefore does not apply).
- [ ] **R2 — reading the verification code (decision D1, taken).** SMTP is
      unset on slots (`mailer.py:42` logs and drops) and the code lives in
      Redis (`auth/router.py:600-604`). Add
      `GET /admin/accounts/{user_id}/email-code` behind `require_admin`
      (the `admin.py:88-269` pattern), answering **404 when
      `ENVIRONMENT == "production"`** so it does not exist where it must
      not. One backend test for the 404, one for the code.
- [ ] **R3 — the eight journeys**, in this order, each its own spec file
      under `frontend/e2e/journeys/`:
      1. **Register → verify.** `/register` with password → onboarding;
         `POST /auth/verify-email` 200 with the code from R2 →
         `email_verified` true on `/auth/me`.
      2. **Session.** Login → dashboard → reload (the refresh cookie
         `sabado_refresh` rotates, `SESSION.md`) → logout → `/dashboard`
         redirects to `/welcome`; the blocklist is honoured (the old
         refresh cookie no longer works).
      3. **Declarative onboarding.** « Pourquoi » → foyer → logement →
         « Passer » on mail → complete, with the bag data of
         `.claude/personas/sacs/lea-tom-primo-mobile.yaml` (already
         prefixed); records exist via `GET /children` and `GET /assets`.
      4. **Documents.** Upload `.claude/fixtures/01_CNI_SPECIMEN.pdf` on
         `/documents`, open it, attach it to the foyer member; the row
         renders, `GET /shared-documents` lists it, « Reliée à » names the
         member.
      5. **Vault, zero-knowledge.** `/vault/setup` passphrase → seal a
         note → lock → unlock → read it back. A network capture over the
         whole journey asserts **only `*_enc` fields leave the browser**;
         `beforeunload` wipes the key.
      6. **Circles.** Create → invite `qa+e2e@example.com` → invitation
         listed → cancel; preview-as-them shows nothing sealed
         (`test_circles.py` semantics on the real stack).
      7. **Calendar round trip.** Create an event → month view →
         `GET /calendar/feed?token=…` returns an ICS containing it →
         import that same ICS URL as a source (`calendar_share.py`,
         self-generated feed, no real calendar).
      8. **Google SSO start only.** `/auth/google/available` → click
         « Continuer avec Google » → the redirect targets
         `accounts.google.com` with the slot's `redirect_uri`
         (`GOOGLE-OAUTH.md`); the callback stays covered by
         `test_google.py`.
- [ ] **R4 — where it runs (decision D2, taken).** `.github/workflows/e2e.yml`
      ssh-runs the suite **on the VM**, the way `deploy-slot.yml` runs
      `slot-deploy.sh` — secrets (`ADMIN_TOKEN`) stay in
      `secrets.slot<n>.env` / `secrets.staging.env`, nothing new in CI —
      triggered by `workflow_run` after `deploy-slot.yml` (against
      `slot<N>.dev.sabado.io`, the slot the run deployed) and as a
      **required step in `deploy-production.yml`** against staging before
      the promotion. Base URL from `E2E_BASE_URL`. Budget: 3 minutes.
- [ ] **R5 — a failure names the journey and keeps the trace.**
      `trace: 'retain-on-failure'`, uploaded as a workflow artifact; the
      step summary lists the eight journeys with pass/fail.
- [ ] **R6 — the account is always removed**, on failure too (a
      `test.afterAll` that runs even when a journey threw), and the suite
      refuses to start if `E2E_BASE_URL` resolves to `app.sabado.io`.

## Files

`frontend/e2e/journeys/01-register.spec.ts` … `08-google-start.spec.ts`
(new) · `frontend/e2e/fixtures/account.ts` (new) ·
`frontend/playwright.config.ts` (from `sabado-25`, one project added) ·
`.github/workflows/e2e.yml` (new) · `.github/workflows/deploy-production.yml`
(one required step) · `backend/app/api/admin.py` (R2) ·
`backend/tests/test_admin.py` (R2) · `deploy/e2e-run.sh` (new, VM side,
mirrors `slot-deploy.sh`'s shape).

## Verify

- [ ] Red test (R2): `GET /admin/accounts/{id}/email-code` → 404 with
      `ENVIRONMENT=production`, the code otherwise; RED today (route absent).
- [ ] Red test: journey 2 against a slot deployed at `8169a600` (main
      before `#1230`) fails on the refresh-rotation case P1-2 describes;
      green on `main`.
- [ ] All eight green against slot 1 on `main`; total under 3 minutes;
      `GET /admin/accounts` no longer lists the account afterwards.
- [ ] Journey 5's capture contains no cleartext note body.
- [ ] A deliberately broken journey (a wrong selector) shows a red step,
      the journey's name, and an attached trace in the run.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.

## Out of scope

Mail connect and ingestion (no throwaway IMAP mailbox per slot; the
persona guardrail exists because the QA account aliases a real Gmail);
« Parler à Sabado » (no AI provider on staging or slots); visual
regression; Firefox and WebKit.
