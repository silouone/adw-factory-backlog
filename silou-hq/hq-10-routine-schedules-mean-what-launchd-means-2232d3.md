---
id: hq-10-routine-schedules-mean-what-launchd-means-2232d3
type: bug
status: in-progress
priority: 2
created: 2026-10-02
depends: []
attempts: []
---
# Routine schedules mean what launchd means: weekdays, every slot, and "at load"

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Stories 25 and 26. Three readings of a plist are wrong today. The Routines widget and the Today widget can disagree about the same routine.

- `src/web/derive.ts` `todayAt` ignores `weekday`. A Sunday-only routine shows **NEXT** on a Friday. Today uses the weekday-aware `nextFire`, so the two widgets give different answers.
- `src/graph/build.ts` keeps only `StartCalendarInterval[0]`, locally and in `m1?.routines` (`r.calendar?.[0]`). A routine scheduled twice a day appears once on the dial.
- A plist with no schedule reads "always on", including one-shot `RunAtLoad` jobs (e.g. `com.user.ollama-env`). The M1 scanner already reports `keepAlive` and `runAtLoad`, but they are dropped.

## Red first

- `routineRows` with a fixed `now` on a Friday and a routine whose `weekday` is Sunday: the routine is neither NEXT nor EARLIER today.
- For one fixed `now` and fixture, the first NEXT in Routines is the same routine and time as Today's first "next up".
- A fixture plist with two `StartCalendarInterval` slots gives two dial stations (or one routine carrying both slots), for both `mbp` and `m1`.
- A fixture plist with `RunAtLoad` and no `KeepAlive` and no schedule reads "at load", not "always on". `KeepAlive` alone still reads "always on".

## Acceptance criteria

- [ ] One schedule meaning feeds both widgets (no second next-fire calculation).
- [ ] Every calendar slot is kept, on both machines.
- [ ] "at load" and "always on" are distinct in the graph meta and in the Routines widget.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- (nothing)
