---
id: hq-07-search-inspector-and-full-screen-in-preact-a2c261
type: feat
status: in-progress
priority: 2
created: 2026-10-01
caps: {minutes: 150, turns: 400}
depends: [hq-05-the-widget-frame-in-preact-d4bbf4]
attempts: []
---
# Search, the inspector and full screen as components

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Port `/` search (grouped by kind; Enter selects and flies), the inspector (kind, cluster,
machines, connections, file preview) and the five full-screen views (factory iframe, project,
output, skills library with filters, file) to Preact. Selecting from any of them still drives
the canvas camera (flight and comet).

## Red first

- Esc order: search closes first, then full screen, then the selection is cleared and the view fits.
- Factory full screen: not reachable shows the honest sentence, with no blank iframe.
- File and output: an id the server rejects shows its message (no stack trace, no raw 404 page).
- Skills library filters: all, portable, claude-only, codex-only and m1 give the expected counts for a fixture graph.

## Acceptance criteria

- [ ] No action button exists anywhere (v1 is read-only); a test asserts there's no button whose label starts with ▶, ■ or ⏸.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-05-the-widget-frame-in-preact-d4bbf4
