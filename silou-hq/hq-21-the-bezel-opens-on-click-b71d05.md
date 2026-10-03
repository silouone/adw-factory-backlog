---
id: hq-21-the-bezel-opens-on-click-b71d05
type: feat
status: in-review
priority: 1
depends: [hq-20-ring-focus-header-tuck-and-trackpad-8c2e47]
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-21-the-bezel-opens-on-click-b71d05-1791011509045","branch":"adw/hq-21-the-bezel-opens-on-click-b71d05","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-21-the-bezel-opens-on-click-b71d05-1791011509045/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/22","provider":"claude","model":"claude-sonnet-5-5","rebased":"81a1631266efaaa06b236cf3031da17785506efb"}]
---
# Clicking a node opens its action bezel at the centre (READ and LINKS)

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

The action bezel, read-only half (spec-v2 → The action bezel; stories 24–33, 35, 15, 74, 75):
- **Transit.** Selecting any node (canvas, link spoke, widget row) glides it to the **exact** screen centre and blooms the dial on settle, or after 650 ms at most.
  - Bloom: scale .55→1 and 32°→0 over 460 ms, sectors staggered by 45 ms.
  - Retract: scale .45, −40°, 200 ms.
- **Geometry** at the UI scale `s = clamp(viewportWidth/3000, 0.7, 1)`:
  - radii R0 44, R1 60, R2 122, HOT 138, SPOKE0 162;
  - four fixed sectors (READ N, OPEN E, CONTROL S, LINKS W, ±43° each); OPEN and CONTROL drawn dashed and empty for now;
  - READ as orbit tracks with upright text;
  - LINKS as radial spokes, 14 per page, 8° apart.
- **LINKS paging.** `TURN_AT 110`, `REST_MS 320`, the dominant axis wins, a 40° turn; `‹ n / N ›` inside the readout pill.
- **Readout pill.** The node's kind, label and one fact; an action's glyph, label and hint on hover.
- **The ember focus zone.** The JS mask, `R = BASE + (reach − BASE)·.45 + 170` with `BASE = HOT+14`, eased .075 per frame; no page dim, border or blur. The material is spec-v2's ember values. No `?lens` switch.
- **Finish.** The hub in the node's ring hue; anodized sectors; sector hues READ `#8f82c4`, OPEN `#4b92b0`, CONTROL `#c3a15a`, LINKS `#7cc5ff`.
- **The pure action catalogue** `sectors(node, routineDetail?, neighbours)`, READ and LINKS entries only; READ opens the existing full-screen views.
- **The clamp relaxes while focused.** `clampView` centres its bound on the focused node, so a later pinch never snaps back.
- **First click.** Pointer input is queued until the first graph load, then replayed once.

## Red first

Pure tests:
- the sector bounds and fixed order; an empty sector is present, not omitted;
- the orbit radii `HOT+30+i·27` and the upright-text flag for the lower half;
- the LINKS page count, the turn accumulator (one notch = one page, cool-down respected), and the dominant axis;
- the zone radius easing converging to its target;
- the UI scale at 1440, 3000 and 4000 px;
- the focused clamp (exact centring is a fixed point of the clamp);
- `sectors()` per kind (memory, skill, cluster, routine with and without output).

## Acceptance criteria

- [ ] Selecting from a widget row, a link spoke or the canvas all lead to the same centred bezel.
- [ ] Esc or a click elsewhere closes it.
- [ ] Under reduced motion there is no bloom rotation.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] In a **visible** tab, bloom and retract timings feel right and the zone size glides between the LINKS and READ sizes.

## Blocked by

- hq-20 (same canvas surface).
