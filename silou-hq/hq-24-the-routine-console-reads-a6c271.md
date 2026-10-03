---
id: hq-24-the-routine-console-reads-a6c271
type: feat
status: done
priority: 1
depends: [hq-21-the-bezel-opens-on-click-b71d05]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-24-the-routine-console-reads-a6c271-1791019319001","branch":"adw/hq-24-the-routine-console-reads-a6c271","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-24-the-routine-console-reads-a6c271-1791019319001/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/24","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The routine console: a read-only deep view of every routine

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 42–47 and 41 (spec-v2 → The routine reader; The routines console):
- **`GET /routine?id=`** for a routine the graph knows:
  - this Mac: plist (parsed and raw XML), resolved script (path, exists, content ≤ 40 000 chars), working directory, schedule, `launchctl` pid, last exit and disabled state, and log tails of 120 lines through the bounded-tail rules;
  - M1: what the snapshot holds.
- **Health** `unloaded | broken | running | failing | ok` with a `why`. It **extends hq-12's "failed"** and never redefines it: `broken` = script missing, `unloaded` = not in launchd or unreadable.
- **Script resolution.** The first `.sh/.py/.ts/.js/.mjs/.rb` argument (absolute against `WorkingDirectory`), or `-m <module>` under the working directory or its `src/`. **No fallback to `ProgramArguments[0]`.** The path must be under `$HOME`, outside `~/Library`, and not the program.
- **The deep view, in Preact + signals.**
  - Opening it: the rings are fitted, then shrink into a clickable corner mini-ring (centre `(158, h−158)·s`, radius `118·s`, 0.6 s); the widgets, zoom controls and bezel fade.
  - Esc or the mini-ring returns and re-selects.
  - The side column: "N need you", health dots, M1 greyed, "◎ back to the rings".
  - The overview: the schedule sentence (a temporary reader is fine; hq-26 brings the codec), the 24 h timeline, the verdict with the missing script named, and log tabs stderr/stdout/plist with **repeated identical lines folded ×N**.
- READ gets **Routine console** for routines. The console toolbar buttons are drawn **disabled** until hq-25.

## Red first

- Pure: health from fixture `launchctl` outputs (consistent with hq-12's fixtures); script resolution, including `-m` modules and **a plist whose only argument is `/opt/homebrew/bin/python3` → no script**; the ×N fold.
- HTTP: `/routine?id=` → 404 outside the allowlist; bounded tails; launchd unreadable → `unloaded` with a `why`, never a 500.
- UI (happy-dom): the console with no routines; with launchd unreadable; an M1 routine read-only.

## Acceptance criteria

- [ ] brain-hub `d1-channels`, `d2-redshift` and `d3-snapshot` show as **broken**, naming their missing `run-*.sh`.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-21 (READ → Routine console).
