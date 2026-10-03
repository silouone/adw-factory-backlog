---
id: hq-28-remove-and-restore-a-routine-92f4c8
type: feat
status: in-progress
priority: 2
depends: [hq-26-edit-a-routine-schedule-c94d2a]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-28-remove-and-restore-a-routine-92f4c8-1791030675559","branch":"adw/hq-28-remove-and-restore-a-routine-92f4c8","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-28-remove-and-restore-a-routine-92f4c8-1791030675559/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/30","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Remove a routine, and restore any routine HQ changed

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 40, 54 (spec-v2 → The write surface → Previous versions):
- **`remove`** (inline confirmation): snapshot, `bootout`, move the plist to `~/.Trash/`. Toast: "Removed".
- **`restore`** (inline confirmation): write back the newest snapshot pair and consume it; `bootstrap` if the plist came back. Toast: "Restored". Nothing to restore → 409.
- **Removed routines stay reachable** through the pseudo-id `routines:removed/<label>`, derived by the server from `cache/previous/` (not from the graph). They are listed in the console's side column, with Restore.
- At most 10 snapshots are kept per label.
- CONTROL gets "Restore previous" when a snapshot exists.

## Red first

The HTTP surface:
- the remove argv and the Trash move;
- **restore on `routines:removed/<label>` after a rebuild whose graph no longer has the routine;**
- restore after a schedule edit puts the old plist back byte-equal;
- 409 when there is no snapshot;
- the 11th snapshot evicts the oldest.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-26 (the snapshot store).
