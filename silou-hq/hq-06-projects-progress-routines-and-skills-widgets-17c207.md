---
id: hq-06-projects-progress-routines-and-skills-widgets-17c207
type: feat
status: queued
priority: 2
created: 2026-10-01
caps: {minutes: 150, turns: 400}
depends: [hq-05-the-widget-frame-in-preact-d4bbf4, hq-03-the-factory-adapter-is-a-contract-a06397]
attempts: []
---
# Projects, In progress, Routines and Skills deck as components

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Port the remaining four widgets onto the hq-05 frame:
- **Projects:** one row per cluster with live, blocked, review and waiting badges, plus "⤢" to full screen.
- **In progress:** running now, waiting on you, ready to review, blocked; empty state.
- **Routines:** a timetable with earlier, next, queued, always-on and on-event rows, plus "output ⤢" where an output exists.
- **Skills deck:** recent, portable and claude-only tabs, with runtime and machine marks.

## Red first

- Routines: given a fixed "now", statuses are earlier, next (exactly one), queued and always on; and "output ⤢" appears only when `hasOutput` is set.
- Skills deck: the tab counts equal the graph's portable and claude-only totals.
- In progress: with nothing to show, the empty-state sentence renders.

## Acceptance criteria

- [ ] All seven widgets are components; the old string renderers are deleted.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-05-the-widget-frame-in-preact-d4bbf4
- hq-03-the-factory-adapter-is-a-contract-a06397
