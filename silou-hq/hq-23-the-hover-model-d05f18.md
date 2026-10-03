---
id: hq-23-the-hover-model-d05f18
type: feat
status: done
priority: 2
depends: [hq-21-the-bezel-opens-on-click-b71d05]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-23-the-hover-model-d05f18-1791019344917","branch":"adw/hq-23-the-hover-model-d05f18","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-23-the-hover-model-d05f18-1791019344917/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/23","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The hover model: magnet, swell, ghost, promote

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 16–23 (spec-v2 → The hover model). It is a **pure state machine** `(state, event, now) → state` with events pointer-move, pointer-leave, click, tick and key; the canvas only draws its state. Constants from the prototype, as decisions:

```ts
MAGNET  = { graceMs: 380, releasePx: 64, lean: 0.24, maxLean: 7, spring: 0.28 } // ring snap-in 140 ms from +9 px
SWELL   = { t0: 300, t1: 950, sigmaPx: 120, wave: "sin(t/480 − d/34)", hoveredScale: 1.3, named: 10 }
GHOST   = { stillMs: 750, maxSpeedPxPerMs: 0.3, popInMs: 180, popFromScale: 0.9, fadeOutMs: 165 }
PROMOTE = { holdRadius: "HOT + 22", brightenMs: 220, hoverCloseMs: 300 }
```

- The contact clock resets on a node change, on speed > 0.3 px/ms, and during the magnet hold.
- A near-miss click during the hold selects the held node.
- No swell inside an open focus zone.
- The ghost pops **in place**, animating its inner element only (never the positioned wrapper); full focus shadow .95, dial lines at .7.
- Moving onto a non-empty sector promotes the ghost to the live bezel in place: no camera move, a real selection.
- A hover-opened bezel closes 300 ms outside the zone; a click-opened one stays.
- Reduced motion: no lean, no wave, no bloom; same states.

## Red first

Scripted event sequences against the pure machine:
- the hold, and release by another node, by distance > 64 px and by timeout;
- a near-miss click during the hold → the held node;
- passing over nodes fast never reaches ghost;
- 750 ms still → ghost, with resets on speed and on a node change;
- ghost → promote on sector entry;
- hover-close at 300 ms vs a click-opened bezel staying;
- the reduced-motion flag zeroes the lean.

## Acceptance criteria

- [ ] Click and hover reach the same bezel UX.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] In a visible tab, the ghost never flies in from a corner, and passing over nodes stays calm.

## Blocked by

- hq-21 (the ghost and promote are the bezel).
