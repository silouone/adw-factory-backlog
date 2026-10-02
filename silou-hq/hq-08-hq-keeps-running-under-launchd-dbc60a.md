---
id: hq-08-hq-keeps-running-under-launchd-dbc60a
type: feat
status: in-progress
priority: 2
created: 2026-10-01
depends: [hq-04-the-landing-answers-in-under-a-second-6ded62]
attempts: []
---
# HQ keeps running under launchd

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Spec: one launchd agent on this Mac with `KeepAlive` and a fixed port, opened from a bookmark.

## What to build

- A LaunchAgent plist template (label `com.silou.hq`) that runs the server with `KeepAlive` and logs to `cache/`.
- It reads `ADW_WEB` from a gitignored local env file.
- `install`, `uninstall` and `status` scripts (package scripts or a justfile).
- A README section.

## Acceptance criteria

- [ ] The plist renders from the template with the repo path and port, and the test checks the rendered plist (KeepAlive true, the program path, no secret in it).
- [ ] The env file is gitignored (test reads `.gitignore`).
- [ ] The operator runs install once; after that, HQ answers on the port after a logout and login (operator-verified, noted in the PR).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-04-the-landing-answers-in-under-a-second-6ded62
