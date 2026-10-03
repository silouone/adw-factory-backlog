---
id: hq-29-create-a-routine-07ad5e
type: feat
status: done
priority: 2
depends: [hq-27-edit-and-recreate-a-routine-script-5e07b3]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-29-create-a-routine-07ad5e-1791033488324","branch":"adw/hq-29-create-a-routine-07ad5e","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-29-create-a-routine-07ad5e-1791033488324/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/33","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Create a new routine from the console

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 55–57 (spec-v2 → The write surface → create validation):
- **+ New routine** opens the editor with Name (prefix `com.silouane.` fixed) and Runs in (a picker over repo nodes present on this Mac).
- **`create` on the pseudo-node `routines:new`:**
  - `label` matches `^com\.silouane\.[a-z0-9][a-z0-9-]{0,47}$`;
  - `runsIn` is a this-Mac `repo` node id;
  - the script goes to `<repo root>/routines/<name>.sh`, the plist to `~/Library/LaunchAgents/<label>.plist`;
  - either existing → 409;
  - logs are always `~/Library/Logs/silou-hq/<name>.{out,err}`;
  - then `bootstrap`.
- Confirmed in the save bar. Toast: "Created".

## Red first

The HTTP surface:
- the label pattern (good, bad, too long, path-shaped);
- `runsIn` not a repo, or an M1 repo → 400 or 403;
- an existing plist or script → 409, with nothing written;
- the written plist parses with the fixed log paths and the codec's schedule keys;
- the script is `+x`.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] Create a throwaway routine, Run now, see its log in the console, then Remove it.

## Blocked by

- hq-27 (the script editor).
