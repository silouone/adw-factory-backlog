---
id: hq-35-today-shows-perso-and-pro-calendars-fbb29b
type: feat
status: blocked
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 400}
depends: [hq-34-accounts-and-hq-connect-for-google-96fc57]
attempts: [{"runId":"hq-35-today-shows-perso-and-pro-calendars-fbb29b-1791033481915","branch":"adw/hq-35-today-shows-perso-and-pro-calendars-fbb29b","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-35-today-shows-perso-and-pro-calendars-fbb29b-1791033481915/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Today shows the day's Google events under a perso / pro lens you flip left and right

> Spec: `~/personal_project/silou-hq/docs/spec-v3-mail-calendar.md` (binding; it amends rule #2 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` still bind otherwise). Rules: `CLAUDE.md`. Read-only toward every provider; `POST /action` stays the only write route. Tokens and feed URLs live only in the Keychain. Sabado's Gmail/Calendar code (`~/personal_project/SABADO/sabado/backend/app/`) is a reference for shapes and traps, not for copying (it is Python).

## What to build

Stories 5–8 and 10 (spec → Calendar), end to end for **Google**:
- **A Google Calendar adapter.** `calendarList` validates the configured ids. Then `events.list` per calendar with `singleEvents=true`, `orderBy=startTime`, and the window from the local start of today to tomorrow 12:00. It skips cancelled events. **All-day events stay a DATE**; timed events are epoch ms. It produces `CalEvent`, never throws, and returns `{ok:false, state, reason}` with the account id and endpoint.
- **A `calendar` source** in the graph store (`refresh.ts`), with a timeout and the last good value, polled every `polling.calendarMin` (15). Per-account state comes from hq-34.
- **`GET /calendar.json`:** `{ lenses: {perso, pro}, accounts: [{id, state, asOf?, fix?}], window }`.
- **The Today widget's calendar section** (it replaces the "isn't connected yet" empty state):
  - the header `‹ perso · pro ›`;
  - rows with time or "all day", title, location and an account dot;
  - past events dimmed; the *now* marker; the *next* event bold with "in N min";
  - each lens's empty state;
  - one fix line per account that is not ok (`bun run hq:connect <id>`).
- **Switching lens:** the buttons, ←/→ while the widget has focus, and a horizontal trackpad swipe over the widget (|deltaX| > |deltaY|, debounced, not propagated to the rings). The lens persists in `localStorage` `hq.calendar.lens`.

## Red first

- Adapter fixtures: a timed event, an all-day event (date kept), a cancelled event (skipped), a calendar id not in `calendarList` (per-account error), 401 → `needs-reconnect`.
- Pure merge and window: grouping by lens, ordering, *now*/*next* at a fixed clock, events crossing midnight, the DST change on 2026-10-25.
- HTTP: `/calendar.json` shape; still GET-only; the leak guard (no token in the response).
- UI (happy-dom): no calendar accounts; an empty lens; switching by button and by key, persisted; one account needing reconnect.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] Operator live check (after hq-32): the perso lens shows today's events from your wife's shared calendar; flipping to pro shows the "not connected" fix line until hq-36.

## Blocked by

- hq-34 (accounts, Keychain, OAuth refresh).
