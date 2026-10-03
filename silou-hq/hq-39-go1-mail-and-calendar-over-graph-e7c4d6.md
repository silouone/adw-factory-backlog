---
id: hq-39-go1-mail-and-calendar-over-graph-e7c4d6
type: feat
status: queued
priority: 2
created: 2026-10-03
caps: {minutes: 180, turns: 500}
depends: [hq-33-go1-consent-probe-and-ics-publish-25bb9b, hq-36-go1-calendar-from-its-published-ics-6b07b5, hq-38-rules-triage-and-category-chips-29ad25]
attempts: []
---
# Go1 mail and the live Go1 calendar arrive through Microsoft Graph

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

## What to build

Spec → Connecting (Microsoft), Calendar, Mail (Graph):
- **OAuth for Microsoft:** a public client, PKCE, loopback `http://localhost:<port>`, scopes `offline_access User.Read Mail.Read Calendars.ReadBasic`, authority and client id from config (recorded in hq-33). **The refresh token rotates on every use; write the new one back every time.** `hq:connect go1` and `hq:disconnect go1` (the latter deletes the Keychain item and prints where to remove the grant).
- **Graph mail adapter:**
  - `/me/mailFolders/inbox/messages` with `$filter=receivedDateTime ge …` and the spec's `$select`;
  - `bodyPreview` truncated to 300;
  - `providerSignals` from `inferenceClassification`, `importance` and `flag`;
  - `listUnsubscribe` from `internetMessageHeaders` **if** the list `$select` accepts it (verify against a live response; otherwise it is false and the rules lean on `outlook:focused`);
  - `link` = `webLink`.
  - VIP from Sent Items.
  - It feeds the same `mail` source, cache and `rules-v1`.
- **Graph calendar adapter:** `/me/calendarView?startDateTime&endDateTime` with `Prefer: outlook.timezone`. Graph expands recurrences. It feeds the pro lens.
- **Config switch:** the `go1` account gains `calendar` in its roles; `go1-cal-ics` is removed. Document this in the README.

## Red first

- OAuth with a fake `fetch`: rotation written back, `invalid_grant` → `needs-reconnect`, `AADSTS65001` (consent required) → `not-connected` with a reason naming admin consent.
- Mail fixtures: focused/other, high importance, flagged, `bodyPreview` truncation, paging via `@odata.nextLink`.
- Calendar fixtures: a recurring occurrence, all-day (date kept), the timezone header honoured.
- `rules-v1` with Graph signals (`outlook:focused` + `outlook:importance-high` → important).
- The leak guard over `/mail.json` and `/calendar.json`.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] Operator live check: `hq:connect go1` succeeds; Go1 important mail appears under the pro chip; the pro lens shows live Go1 meetings with no "as of" lag.

## Blocked by

- hq-33 (consent granted, client id recorded). **Do not dispatch before hq-33 records consent.**
- hq-36 (the pro lens it replaces).
- hq-38 (the rules it feeds).
