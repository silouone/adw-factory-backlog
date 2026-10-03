---
id: hq-27-edit-and-recreate-a-routine-script-5e07b3
type: feat
status: done
priority: 1
depends: [hq-26-edit-a-routine-schedule-c94d2a]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-27-edit-and-recreate-a-routine-script-5e07b3-1791030627974","branch":"adw/hq-27-edit-and-recreate-a-routine-script-5e07b3","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-27-edit-and-recreate-a-routine-script-5e07b3-1791030627974/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/31","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Edit a routine's script, or recreate a missing one, with a live diff

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 46, 50–53 (spec-v2 → The routines console → Editor; The write surface):
- **"What it runs":** a code editor with a gutter, a bash highlighter (comments, strings, `$vars`, keywords) and Tab = 2 spaces; a **live LCS diff** below it with +/− counts.
- **A missing script** is pre-filled with a template: shebang, `set -euo pipefail`, `cd` into the working directory. The verdict's **Recreate the script** opens it.
- **The `script` verb** takes `{ content }` only:
  - the target is the plist-resolved script (hq-24's rules, no fallback; no resolvable script → 403);
  - ≤ 40 000 characters → else 400;
  - snapshot first; keep the file's mode, `+x` when new;
  - destructive, so the inline confirmation applies.
- The ledger stores the content's length and sha256, **never the content**. Toast: "Script saved".

## Red first

- Pure: the LCS diff and its counts (insert, delete, replace, empty ↔ full); the template given a working directory.
- HTTP: a payload `path` is ignored; an interpreter-only plist → 403; the size cap → 400; the snapshot precedes the write; the ledger line has no content; a new file gets `+x`.

## Acceptance criteria

- [ ] **Demo:** recreating brain-hub's `d2-redshift` script from the console turns its health from broken to ok after a Run now (the operator supplies the script body).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-26 (the editor shell, save bar and snapshot store).
