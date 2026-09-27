---
id: adw-fe-23-zoom-and-scroll-the-run-timeline
type: feat
status: done
priority: 2
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: [adw-fe-22-the-gantt-axis-is-not-time, adw-render-04-the-run-screen-becomes-components]
attempts: [{"runId":"adw-fe-23-zoom-and-scroll-the-run-timeline-1790517678918","branch":"adw/adw-fe-23-zoom-and-scroll-the-run-timeline","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-23-zoom-and-scroll-the-run-timeline-1790517678918/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/125","provider":"claude","model":"sonnet"}]
---
# The Gantt can only ever be as wide as the window, so a crowded run has nowhere to go

> **Unblocked 2026-09-20 (`adw-render-05-parity-sweep`, R5).** Both
> dependencies are `done`: `adw-fe-22` and `adw-v1.11`'s **P3**
> (`adw-render-04-the-run-screen-becomes-components`, the run screen becomes
> components). `render-run.ts` — the module this ticket's blast radius and
> its original R4 were written against — no longer exists; the run screen's
> Gantt lanes are `src/web/ui/Lanes.tsx` now.
>
> **R4 struck below, not reimplemented.** R4 asked for the
> `innerHTML`-save-and-restore band-aid a fifth time (after `adw-usage-01` PR
> #61, the usage chip, and the dock scope). Under the component run screen,
> zoom and `scrollLeft` are component state and simply survive a live tick —
> a `/run-events` frame updates props, it never replaces `#run-console` with
> a fresh `innerHTML`. There is nothing to capture and restore, and
> reimplementing the capture/restore dance would be dead code satisfying a
> requirement the architecture already satisfies structurally.

> **Spec authority:** `specs/adw-v1.10-linear-timeline.md` §"Solution" item 3
> and §"Why `fit` is the default". Operator ask, verbatim: *"ENABLE horizontal
> scroll on the gantt IF IN SOME CASE it does NOT fit inside the view."*
>
> **Depends on `adw-fe-22`**, which makes the axis linear and turns instant
> nodes into markers. Do not start this before that is `done`: zooming a
> warped axis magnifies the distortion instead of relieving it.

## Why this exists, and why it is not the main fix

The run's dynamic range is `25m 51s : 1ms` ≈ **1.55 × 10⁶ : 1**. No zoom makes
a 1ms node a readable box — that would need a ~40-billion-pixel track. `fe-22`
is what actually fixes legibility, by refusing to spend width on instant
nodes; measured over 122 boxes in 28 banked runs, it takes the median box from
100px to 177px and full-label legibility from 31% to 58% **at zoom 1, with no
scroll at all**.

This ticket adds the valve for the rest. Measured, auto-zooming until *every*
box shows its full label needs a median **2.6×** at a 900px track — which is
why `fit` is the default and zoom is a control the operator reaches for, not a
policy applied on their behalf.

## Requirements

- [ ] **R1 — one scroller, and the lane names do not move.** The time axis
      (`.axis-row`'s `.axis`), the gridlines (`.grid-overlay`) and **every**
      `.ln-track` share a single horizontal scroll container, so they scroll
      together as one chart. The `.ln-id` column (lane name, model, `share`
      meter) stays fixed and readable at every scroll position.
- [ ] **R2 — zoom widens the track, not the page.** A `--zoom` multiplier
      sets the scrolled content width (`calc(100% * var(--zoom))`). The page
      itself never scrolls horizontally; only the chart does.
- [ ] **R3 — `fit` is the default.** First render is zoom `1`: the whole run
      on one screen, at true proportions, no scrollbar. The control offers at
      least `fit` / `2×` / `4×`.
- [ ] ~~**R4 — zoom and scroll survive a live tick.**~~ **Struck** — satisfied
      structurally by `adw-render-04`. Zoom and `scrollLeft` are component
      state on `<Lanes/>`; a live tick updates props, it never replaces
      `#run-console`'s `innerHTML`, so there is nothing to capture or
      restore.
- [ ] **R5 — a scrolled chart still says where it is.** While the chart is
      scrolled, the operator can still tell which part of the run is in view.
      The `share` meters in the fixed `.ln-id` column plus the time ticks
      scrolling with the chart are sufficient; an overview minimap is
      **optional** and explicitly not required to close this ticket.
- [ ] **R6 — clicking still works at every zoom.** A box or marker click
      still routes through `data-rail` to the panel (`showScope`), and the
      `.sel` highlight still lands on the right element, at any zoom and
      scroll offset. The existing delegated click handler must keep working —
      it keys off `data-rail`, never position, so this is a regression guard,
      not new work.

## Build protocol

Article I still binds, but note honestly what is and is not testable here:
this is a DOM/CSS container change and `timeline.ts`'s own stance already
says *"pixels are not a test"*. Do not fabricate pixel assertions.

1. **Red tests** for the parts that are pure and real:
   - `renderLanes`'s HTML places the axis, the grid overlay and every
     `.ln-track` inside one scroll container element, and the `.ln-id`
     column outside it — assert on the emitted structure, the same way
     `test/web/render-run.test.ts` already asserts on raw HTML.
   - The zoom control is rendered, defaults to `fit`, and emits the `--zoom`
     hook at its documented default of `1`.
   - R6: every drawn box and marker still carries a `data-rail` attribute
     matching its rail entry (`railKeyOf`) — the existing rail/Gantt identity
     contract, re-asserted against the new markup.
2. **Confirm red.** Present for operator review before any source.
3. Implement to green.
4. `bun run lint && bunx tsc --noEmit && bun test` — all green.

## Blast radius — read before estimating

> **Pre-migration note (2026-09-20):** the paths below (`render-run.ts`,
> `renderLanes`, `render-run.test.ts`) predate `adw-v1.11`'s P2/P3 and no
> longer exist — the run screen's lanes are `src/web/ui/Lanes.tsx` now,
> tested under `test/web/ui/Lanes.test.tsx`. Left as-is rather than
> re-estimated: re-scoping the blast radius against the component
> architecture is this ticket's own build work, not this chore's.

- `src/web/render-run.ts` — `renderLanes`'s markup is restructured, not just
  restyled. Today `.ln-id` is a flex child **inside** each `.ln` row
  (`:853`), `.axis-row` carries its own `.ln-id` spacer (`:842`), and
  `.grid-overlay` is absolutely positioned at `left:210px` **inside**
  `.lanes` (`:851`). A two-column layout (fixed id column ‖ one scroller)
  replaces all three of those arrangements; the hard-coded `210px` appears in
  two places and in the `@media(max-width:900px)` override at `:1040`, which
  narrows it to `140px`.
- `src/web/server.ts` — the `/run-events` route composes
  `renderRunHeader + renderLanes + renderUpNext`. **No signature change
  expected**, but the frame's `lastSent` comparison means an unchanged frame
  is not resent, so the restore path must be correct for the frames that
  *are* sent.
- `test/web/render-run.test.ts` — `chartOnly` and `blocksOf` parse the lane
  markup by regex and will need updating for the new container structure.

## Out of scope

- The linear axis and markers — `adw-fe-22`, which this depends on.
- Vertical (lane) virtualisation. Lane counts are small and bounded.
- An overview minimap (see R5 — optional, not required).
- Pinch/trackpad gesture zoom. The control is enough; gestures can follow if
  the operator asks.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] At default `fit`, a run's chart shows no horizontal scrollbar and the
      whole run is visible.
- [ ] At `2×`, the chart scrolls horizontally, the lane names stay put, and
      the gridlines stay aligned to the boxes they sit behind.
- [ ] **Operator-executed:** with a run live, zoom to `2×` and scroll right,
      then confirm the zoom and scroll position hold across at least two SSE
      ticks. The agent cannot run this — it closes on the operator's word,
      recorded in this ticket.
