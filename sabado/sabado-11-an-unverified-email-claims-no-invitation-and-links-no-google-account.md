---
id: sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account
type: bug
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 180, turns: 500, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account-1789850416702","branch":"adw/sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account-1789850416702/workspace","outcome":"blocked"},{"runId":"sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account-1789859132177","branch":"adw/sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-11-an-unverified-email-claims-no-invitation-and-links-no-google-account-1789859132177/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1220","provider":"claude","model":"sonnet"}]
---
# fix(auth): an unverified e-mail claims no circle invitation and links no Google account

> **Audit:** **P0-1** (axis D), re-verified by the synthesis against the
> code. Held out of the LANES batch as "security"; included here because it
> is a live cross-user read with a two-line fix and two named tests — exactly
> what the bug lane exists for. `review: false`: every Verify bullet is a red
> test or a gate.

## What happens today

`backend/app/auth/router.py:372-375` signs a new account in immediately —
"the account is fully usable before the address is verified".
`auth/dependencies.py:49-86` never checks `email_verified`; outside
`router.py:577` (verify-email itself) nothing does.
`backend/app/api/circles.py:256-266` lists pending invitations by
`invited_email == user.email` **including `invite_token`**; `:277-283`
accepts on the same match.

Sequence: `POST /auth/register {email: <an invited address>}` →
`GET /circles/invitations` → `POST /circles/invitations/accept {token}` →
`readable_user_ids` now returns every member who exposed children, budget,
calendar, assets or documents in that circle. Precondition: the invited
address has no Sabado account yet — the normal case.

Same root, second consequence: `_login_or_link` (`router.py:806-819`) links
a Google `sub` to an existing password account **by e-mail alone**. The
real owner signing in with Google later lands in an attacker-created account
whose password the attacker knows.

`tests/test_circles.py:83` covers the wrong-e-mail case. Nothing covers the
unverified case.

## Requirements

- [ ] **R1** `list_my_invitations` and `accept_invitation` raise `403` when
      `not user.email_verified`. The detail names the reason in the house
      register (French, one sentence, like the 188 existing `detail`s).
- [ ] **R2** `_login_or_link`, case "password account exists for this
      e-mail": when that account is unverified, do **not** link and do not
      create a second account; answer with an explicit "verify first"
      outcome the front already knows how to show (a redirect carrying an
      error code, the way the other callback failures do).
- [ ] **R3** `BACKEND.md` gains the rule, one line: *a query that matches on
      `User.email` (or `invited_email`) as an identity must require
      `email_verified`.* (The CI grep enforcing it is Lane I's,
      `sabado-23`.)

## Files

`backend/app/api/circles.py` · `backend/app/auth/router.py` (**`_login_or_link`
only** — `sabado-21` owns line 879 and `_issue_tokens`; do not touch them) ·
`backend/tests/test_circles.py` · `backend/tests/test_google.py` ·
`.claude/skills/sabado-project/BACKEND.md`.

## Verify

- [ ] Red test: `test_circles.py::test_an_unverified_account_sees_and_accepts_no_invitation`
      — register without verifying, invite that address from another user,
      assert `GET /circles/invitations` is 403 and `POST …/accept` with the
      real token is 403 and the membership does not exist. RED today.
- [ ] Red test: `test_google.py::test_google_does_not_link_to_an_unverified_password_account`
      — an unverified password account and a Google identity with the same
      e-mail: the callback does not set a refresh cookie, no `google_sub` is
      written on the account. RED today.
- [ ] The existing wrong-e-mail test (`test_circles.py:83`) and the whole
      ACL matrix in `test_circles.py` stay green.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

The generation-0 refresh token (`router.py:879`, `sabado-21`); the CI grep
(`sabado-23`); requiring verification for anything else.
