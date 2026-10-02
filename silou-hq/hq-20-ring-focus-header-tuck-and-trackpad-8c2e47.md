---
id: hq-20-ring-focus-header-tuck-and-trackpad-8c2e47
type: feat
status: queued
priority: 1
depends: [hq-19-the-rings-make-room-3f1a92]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: []
---
# Focus a ring, tuck the header, and move like a Mac trackpad

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

Stories 9–14 (spec-v2 → Landing changes → Ring focus, Header tuck, Trackpad):
- **Ring focus.** Clicking a ring's line or band (not a node) focuses it:
  - focusable rings: runtimes, plugins·mcp, skills, memory, sessions, hooks, routines, projects;
  - the veil is one radial gradient (darkness .6, clear over [r0, r1], fade .03 U on thin rings and .05 U on wide ones), drawn last;
  - a tint, no border; 320 ms fades;
  - items named when there are 60 or fewer, or when k > 1.8;
  - the title `TITLE · n — click the ring again or press esc`;
  - released by the same ring, Esc, or any selection.
- **Header tuck.** It tucks at k > 1.15 and untucks below 1.08. Search collapses to a ⌕ tab.
- **Trackpad.** Pinch (`ctrlKey` + wheel) zooms at the cursor by `exp(−clamp(deltaY, ±50)·0.012)`; a two-finger move pans 1:1. A mouse wheel (`deltaMode ≠ 0`, or `deltaX = 0` with `wheelDeltaY % 120 = 0`) still zooms. Pan keeps the rings' centre within 8–92 % of the screen.

## Red first

Pure tests:
- the ring hit-test (a radius → a focusable ring or none);
- the veil's gradient stops for a thin and a wide ring;
- the tuck hysteresis (1.15 up, 1.08 down, no flapping in between);
- the wheel classifier (pinch / pan / mouse) on recorded event shapes;
- the pinch factor clamp;
- the 8–92 % pan bound.

## Acceptance criteria

- [ ] The veil is never drawn as rect + evenodd arcs.
- [ ] A focused ring is released by Esc, by clicking it again, and by selecting a node.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] Pinch and two-finger pan on the Mac trackpad feel native (momentum, natural direction).

## Blocked by

- hq-19 (same canvas surface; serialised to avoid conflicting PRs).
