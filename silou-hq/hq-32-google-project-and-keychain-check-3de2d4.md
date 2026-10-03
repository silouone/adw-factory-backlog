---
id: hq-32-google-project-and-keychain-check-3de2d4
type: manual
status: queued
priority: 1
created: 2026-10-03
depends: []
attempts: []
---
# Operator: a Google project HQ owns, and proof the Keychain is readable under launchd

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

Operator-executed. Nothing here is code. It unblocks the **live** checks of hq-34, hq-35 and hq-37; their factory builds don't wait for it.

## Steps

- [ ] Create a GCP project dedicated to HQ (never sabado's). Enable the **Gmail API** and **Google Calendar API**.
- [ ] Google Auth Platform: set the app name (no logo; a logo forces verification), audience **External**, status **Testing**. Add every Gmail account HQ will connect under **Audience → Test users**. The refresh token lasts 7 days, so re-login is weekly (spec v3 D9). Publishing would need homepage and privacy URLs and is optional.
- [ ] Create an OAuth client of type **Desktop app**. Store its id and secret in the Keychain as `hq.google.client` (`security add-generic-password -s hq.google.client -a client_id -w …` and `-a client_secret`). The exact item names follow hq-34; adjust once it lands.
- [ ] Ask your wife to share her Google calendar with your personal Gmail, set to **"See all event details"**. Note its calendar id (her address) for `hq.config.json` → `calendars`.
- [ ] **Keychain under launchd (spec: Connecting → Keychain under launchd).**
  - Add a dummy item `hq.probe`.
  - Temporarily make a LaunchAgent (or `launchctl asuser`) run `security find-generic-password -s hq.probe -w`, then log out and back in.
  - Record whether it reads. If it doesn't, HQ shows accounts as `locked`; note the finding here, and open a follow-up if an ACL (`-T`) on the item is needed.
- [ ] Record the outcomes below.

## Outcomes

- (fill in)
