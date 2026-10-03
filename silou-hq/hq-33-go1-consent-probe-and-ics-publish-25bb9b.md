---
id: hq-33-go1-consent-probe-and-ics-publish-25bb9b
type: manual
status: queued
priority: 1
created: 2026-10-03
depends: []
attempts: []
---
# Operator: publish the Go1 calendar and find out whether Go1 lets HQ read mail

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

Operator-executed. Go1 mail is Microsoft 365 (`go1.com` MX = `go1-com.mail.protection.outlook.com`). Microsoft's default consent policy excludes `Mail.Read` and `Calendars.Read*` from user consent (spec, Further Notes). This ticket finds out which case Go1 is in. hq-36 needs step 1. hq-39 needs steps 2–4.

## Steps

- [ ] **1. ICS (unblocks hq-36's live check).** Outlook on the web → Calendar → Settings → Shared calendars → **Publish a calendar** → your calendar, "Can view all details" (or titles and locations) → copy the **ICS** link. If publishing is disabled by the tenant, record that: the pro lens then waits for hq-39.
- [ ] **2. Entra app.** Register a public-client app ("Mobile and desktop applications", redirect `http://localhost`), with supported accounts **"Accounts in any organizational directory"**. It can live in a free tenant of yours, or Go1 IT may prefer to register it in the Go1 tenant. Record the client id and authority.
- [ ] **3. Consent probe.** Open the authorize URL in a browser:
  `https://login.microsoftonline.com/organizations/oauth2/v2.0/authorize?client_id=<id>&response_type=code&redirect_uri=http://localhost&scope=offline_access%20User.Read%20Mail.Read%20Calendars.ReadBasic&code_challenge=<any>&code_challenge_method=plain`
  Sign in with your Go1 account and record exactly what the consent screen says: **Accept** (user consent allowed) or **"Need admin approval"**.
- [ ] **4. If admin approval is needed:** file the request (with the request-approval button if the tenant has the admin-consent workflow, otherwise ask Go1 IT). Ask for `Mail.Read` and `Calendars.ReadBasic` together. State that the data stays on your Mac, with no third party.
- [ ] Record the outcomes below. hq-39 stays blocked until consent is granted.

## Outcomes

- (fill in)
