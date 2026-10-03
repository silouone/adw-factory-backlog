---
id: hq-25-run-pause-resume-and-stop-a-routine-1b8e64
type: feat
status: in-review
priority: 1
depends: [hq-22-finder-and-vs-code-are-the-first-real-write-4e9a3c, hq-24-the-routine-console-reads-a6c271]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-25-run-pause-resume-and-stop-a-routine-1b8e64-1791024745620","branch":"adw/hq-25-run-pause-resume-and-stop-a-routine-1b8e64","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-25-run-pause-resume-and-stop-a-routine-1b8e64-1791024745620/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/27","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Run, pause, resume and stop a routine on this Mac

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 37–39, 41 (spec-v2 → The write surface, the action catalogue):
- Verbs, through the injected executor (argv only, `gui/<uid>/<label>`):

  | Verb | Command |
  |---|---|
  | `run` | `launchctl kickstart` |
  | `pause` | `launchctl disable` then `bootout` |
  | `resume` | `launchctl enable` then `bootstrap gui/<uid> <plist>` |
  | `cancel` | `launchctl kill SIGTERM` (destructive: inline confirmation) |

- The executor runs with a timeout and returns `{exitCode, stderrTail}`. The ledger records the outcome `done | failed`.
- **HQ's own label `com.silou.hq`** refuses every control verb; CONTROL shows a disabled "This is HQ".
- **M1 routines** → 403; CONTROL shows "Read-only on the M1".
- **Per-label serialisation:** a second write on a label in flight → 409.
- The CONTROL sector and the console toolbar go live:
  - Run now;
  - Pause or Resume, by disabled state;
  - Stop this run, only while running.
- Toasts: Started, Paused, Resumed, Stopped. The console re-reads `/routine?id=` after each action.

## Red first

The HTTP surface with a recording executor:
- the exact argv per verb;
- `com.silou.hq` → 403 for run, pause, resume and cancel;
- M1 → 403;
- the 409 on a concurrent write;
- a failed command → `outcome: failed` with stderrTail in the ledger.

Pure: `sectors()` for running, idle, disabled, HQ and M1 routines.

## Acceptance criteria

- [ ] Stop is offered only while the routine has a PID, and needs an inline confirmation.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] One real run, pause, resume on a throwaway `com.silouane.*` routine. `cache/actions.jsonl` shows all three.

## Blocked by

- hq-22 (the write surface), hq-24 (the console and routine reader).
