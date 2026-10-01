---
id: hq-05-the-widget-frame-in-preact-d4bbf4
type: feat
status: queued
priority: 2
created: 2026-10-01
caps: {minutes: 150, turns: 400}
depends: [hq-01-buildgraph-is-pure-and-tested-0aabb3]
attempts: []
---
# Widgets become Preact components: the frame, arrange mode, Today, Factory, Mail

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

The prototype's widgets are innerHTML strings. Port the **frame** to Preact + signals (the same
stack as adw web):
- header, actions, body;
- arrange mode: move and resize, 8px snap, persisted layout, reset, default layout follows the window until arranged.

Port three widgets onto it: **Today** (clock, quarter-week grid, next routines, calendar "not
connected"), **Factory** (counts, recent-runs strip, "open ⤢"; not-configured and not-reachable
states) and **Mail** ("not connected"). The rings canvas is untouched.

## Red first (happy-dom; prior art: adw-factory `test/web/*`)

- Factory: reachable shows four counts; not configured and not reachable each show their own sentence, with no counts.
- Mail and calendar show their "not connected" copy, with no fake data.
- Arrange mode off: dragging a header does nothing. On: it moves, snaps to 8, and persists.

## Acceptance criteria

- [ ] No widget renders with innerHTML strings any more (for these three).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-01-buildgraph-is-pure-and-tested-0aabb3
