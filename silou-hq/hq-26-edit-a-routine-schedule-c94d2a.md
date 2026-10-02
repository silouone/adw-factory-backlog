---
id: hq-26-edit-a-routine-schedule-c94d2a
type: feat
status: queued
priority: 1
depends: [hq-25-run-pause-resume-and-stop-a-routine-1b8e64]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: []
---
# Edit a routine's schedule with the 24 h dial, saved with a snapshot

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 44, 48, 49, 52 (spec-v2 → The schedule codec; The routines console → Editor, Save bar):
- **The pure schedule codec**, reusing hq-10's launchd semantics (one definition only):

  ```ts
  // from the prototype's editor model
  type Schedule =
    | { kind: "daily"; at: HHMM }                        // StartCalendarInterval {Hour, Minute}
    | { kind: "days"; at: HHMM; weekdays: Weekday[] }    // one calendar dict per weekday
    | { kind: "every"; minutes: number }                 // StartInterval = minutes·60
    | { kind: "always" };                                // KeepAlive true
  ```

  Functions: `toPlistKeys`, `fromPlist`, `sentence`, `nextRun(schedule, now)`, with a 5-minute snap. A plist it can't express opens read-only ("edit the plist to change this").
- **The editor, "When it runs":** a segmented control, the 24 h dial knob (drag, 5-minute snap, arrows ±5), weekday pills, the sentence plus the next run.
- **The save bar:** the change summary, Discard, Save ⌘S, then an inline "Save and reload".
- **The `schedule` verb.** Snapshot the current plist and script to `cache/previous/<label>/<epochMs>.{plist,script}`, rewrite only the schedule keys, then `bootout` + `bootstrap`. Toast: "Schedule saved".
- The overview sentence switches to the codec.

## Red first

- Pure: the plist ↔ schedule round-trip for all four kinds; inexpressible plists (several times a day, Month/Day keys) flagged; `sentence` and `nextRun` at fixed clocks (before and after today's slot, a weekday wrap); the snap. Shares fixtures with hq-10.
- HTTP: the snapshot is written **before** the plist; only schedule keys change (the rest of the plist is byte-equal after a parse); the argv sequence.
- UI: the save bar is hidden with no changes and shows the summary after a change.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] Dragging the dial knob, and arrow keys, feel right.

## Blocked by

- hq-25 (the control verbs, serialisation and the executor's real path).
