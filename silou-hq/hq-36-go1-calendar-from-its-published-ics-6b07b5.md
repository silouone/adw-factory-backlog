---
id: hq-36-go1-calendar-from-its-published-ics-6b07b5
type: feat
status: in-progress
priority: 2
created: 2026-10-03
caps: {minutes: 150, turns: 400}
depends: [hq-35-today-shows-perso-and-pro-calendars-fbb29b]
attempts: []
---
# The pro lens reads the Go1 Outlook calendar from its published ICS link, and says how fresh it is

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

## What to build

Story 9 (spec → Calendar → ICS; "Why published ICS can lag"):
- **Spike first, and record the result in the PR body.** Can `node-ical` (`expandRecurringEvent`) and `ical.js` run under Bun? Run each on the fixtures below. Pick one and wrap it so the rest of HQ never imports it.
- **An ICS adapter: text → `CalEvent[]` within the window.** It expands RRULE, applies EXDATE and RECURRENCE-ID overrides, and resolves VTIMEZONE, **including Windows zone names** (`Romance Standard Time`), which Outlook emits. All-day events stay a DATE.
- **Fetching:** the URL is read from the Keychain (`hq.ics.<id>`) and **never logged, served or ledgered**. Send `If-None-Match` / `If-Modified-Since` when the server gave validators.
- **The freshness log:** append `{fetchedAt, lastModified, etag, contentHash}` per fetch to `cache/ics-freshness.jsonl`. `asOf` for the account is `Last-Modified`, or the time the content hash last changed. The pro lens header shows "as of HH:MM".

## Red first

- ICS fixtures:
  - a weekly RRULE with an EXDATE;
  - a moved occurrence (RECURRENCE-ID);
  - an all-day event;
  - a Windows TZID;
  - an event across the DST change on 2026-10-25;
  - a malformed file (→ `{ok:false}` naming the account, never a throw).
- Freshness: `asOf` from Last-Modified; from the hash change when there is no Last-Modified; 304 keeps the last value.
- The leak guard: the feed URL appears in no response and no log line.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] Operator live check (after hq-33 step 1): `bun run hq:connect go1-cal-ics` stores the URL; the pro lens shows today's Go1 meetings with "as of". After a day, `cache/ics-freshness.jsonl` gives the measured regeneration lag. Record it in hq-33's outcomes, since it decides how urgent hq-39 is.

## Blocked by

- hq-35 (the calendar source, `/calendar.json` and the lens UI it plugs into).
