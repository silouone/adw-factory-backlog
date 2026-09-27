---
id: sabado-15-recurring-events-keep-their-paris-wall-clock
type: bug
status: done
priority: 2
created: 2026-09-19
review: false
caps: {minutes: 180, turns: 500, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-15-recurring-events-keep-their-paris-wall-clock-1789850667705","branch":"adw/sabado-15-recurring-events-keep-their-paris-wall-clock","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-15-recurring-events-keep-their-paris-wall-clock-1789850667705/workspace","outcome":"blocked"},{"runId":"sabado-15-recurring-events-keep-their-paris-wall-clock-1789859141495","branch":"adw/sabado-15-recurring-events-keep-their-paris-wall-clock","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-15-recurring-events-keep-their-paris-wall-clock-1789859141495/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1219","provider":"claude","model":"sonnet"}]
---
# fix(calendar): recurring events keep their Paris wall-clock across the DST switch, and all-day events keep their date

> **Audit:** **P1-23** (axis N) + the all-day convention (N P2) — Lane C.
> **Deadline: 2026-10-25**, the next DST switch. Re-verified by the
> synthesis (`ics.py:106-111`). `review: false`: hard-gated.

## What happens today

`backend/app/core/ics.py:106-111`: `start = _as_utc(start_at)` then
`rrulestr(recurrence…, dtstart=start)`. dateutil expands in the zone of
`dtstart` — UTC — so a weekly rule keeps the same UTC hour across the last
Sunday of October. `backend/app/api/calendar.py:329`: `zone =
_as_utc(ev.start_at).tzinfo` — always UTC, despite the comment "the ROW's
own zone". `calendar_events` has no zone column (`models/normal.py:72-155`);
the product zone is Europe/Paris (`ai/prompts.py:17`,
`dashboard_deadlines.py:297-305`). `grep -rn 'DST\|Europe/Paris'
backend/tests/test_calendar*.py` → 0.

**Effect:** « tous les lundis 9h » created in September renders at 08:00
from 26 October — in the agenda, the circle iCal feed and the AI briefing.

All-day events are stored under two conventions: the UI sends local
midnight converted to UTC (`EventModal.tsx:313`), imports and the AI write
`00:00Z`. The ICS feed and the chat show the UI-created ones a day early.

## Requirements

- [ ] **R1** Expansion happens in Europe/Paris:
      `dtstart=start.astimezone(ZoneInfo("Europe/Paris"))`, every
      occurrence converted back to UTC before it leaves the function. The
      single-gate docstring stays true — every caller still goes through it.
- [ ] **R2** `calendar.py:329` uses the same zone (or the same helper),
      not the row's UTC tzinfo.
- [ ] **R3** On write, when `all_day` is set, the stored instant is `00:00Z`
      of the event's **Paris date** (a local-midnight instant received from
      the UI resolves to the date it was meant as, not the day before).
      The ICS export emits `DTSTART;VALUE=DATE:` / `DTEND;VALUE=DATE:` for
      all-day rows.
- [ ] **R4** `EventModal.tsx` is touched only if the backend cannot
      normalise without it. Prefer not to.

## Files

`backend/app/core/ics.py` · `backend/app/api/calendar.py` ·
`backend/tests/test_calendar_recurrence.py` ·
`backend/tests/test_calendar_feed.py` (the all-day ICS assertion) ·
`frontend/src/pages/calendar/EventModal.tsx:313` only under R4.

## Verify

- [ ] Red test: `test_calendar_recurrence.py::test_a_weekly_series_keeps_its_paris_wall_clock_across_the_october_switch`
      — weekly Monday 09:00 Paris starting 2026-10-12; the occurrences on
      2026-10-19 and 2026-10-26 are both 09:00 in Europe/Paris (07:00Z then
      08:00Z). RED today.
- [ ] Red test: `test_calendar_feed.py::test_an_all_day_event_exports_its_date`
      — create an all-day event through the API with a Paris local-midnight
      instant; the feed carries `DTSTART;VALUE=DATE:<that date>`. RED today.
- [ ] Red test: the same weekly rule through the ICS feed and through the
      briefing's deadline computation gives the same Paris hour on both
      sides of the switch.
- [ ] Every existing test in `test_calendar_recurrence.py`,
      `test_calendar_feed.py` (11) and `CalendarPage.test.tsx` (117) stays
      green unchanged.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

A per-event zone column; the `CalendarPage` model hook (§6 #7); Google
calendar import semantics.
