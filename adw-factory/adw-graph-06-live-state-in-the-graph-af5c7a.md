---
id: adw-graph-06-live-state-in-the-graph-af5c7a
type: feat
status: queued
priority: 2
created: 2026-09-28
review: true
caps: {minutes: 120, turns: 450, stallMinutes: 20}
depends: [adw-graph-03-the-layered-graph-component-aae152]
attempts: []
---
# The graph shows what is moving and what is stuck

> **Spec:** `specs/adw-v1.14-backlog-tab.md` §10, 2026-09-28, **G5** (the
> `running`, `blocked` and `heldBy` data are computed by `adw-graph-02`).
>
> The operator:
> > "can we enhance the way it display what is currently in progress ? …
> > moving like arrow and some kind of overlay on the active card and or the
> > block cards"
>
> They then refined it:
> > "the dot on the dimmed arrow should be dimmed as well"
>
> **Visual reference only:**
> `/Users/silouane/personal_project/adw-factory/.proto/backlog-graph/`
> (`liveOf`, `LiveSummary` and the `.gn-fx` / `.ge.flow` / `.ge-dot` rules in
> `graph.css`).

## Requirements

- [ ] **R1 — red first.** DOM tests over `<DepGraph/>` (happy-dom) cover every
      class and element below, before any CSS.
- [ ] **R2 — running cards** (`state` `running` or `in-flight`):
      - a live-blue border and glow;
      - a light diagonal sweep overlay (≈2.4 s loop) in its own
        `pointer-events:none` layer;
      - a spinning `◌` glyph.
      - **The status text:** `running` or `in flight`, plus `· 12m` elapsed
        when the row has `lastAt`. The elapsed time comes from the `now` prop;
        components never read the clock (`src/web/ui/README.md`).
- [ ] **R3 — flow.**
      - An edge out of a running node is `flow`: animated live-blue dashes.
      - It carries two moving dots (`animateMotion` along the edge's own path,
        staggered by half a cycle) toward what it unblocks.
      - **A dot takes its edge's dim / lit state**, so a dimmed edge has
        dimmed dots.
- [ ] **R4 — blocked cards:**
      - red diagonal hatching over an **opaque** base (G7);
      - a slow red pulse;
      - status `blocked`;
      - the tooltip carries `row.reason` (as served by the projection since
        adw-backlog-03).
- [ ] **R5 — held.**
      - Every card with `heldBy` shows a quiet `⛔ via 01` badge.
      - Edges out of a blocked or held node are `stalled`: red, dotted, still.
- [ ] **R6 — header summary** in the drawer (`adw-graph-04`) or above the
      graph, whichever is merged: `◌ 3 running · ✕ 1 blocked · ⛔ 10 held
      behind a block`. Omit a part whose count is zero.
- [ ] **R7 — `prefers-reduced-motion: reduce`** stops every animation and
      hides the dots. The static states (glow, hatching, badges) remain.
- [ ] **R8 — no regression of G7.**
      - The overlays live inside the card.
      - The "no edge sample inside a card" test from `adw-graph-03` stays
        green.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` →
Backlog:
- **cmc**, with its three running tickets: glowing cards and dots flowing
  toward 07/08/10/11/12. Hover 07: only the 06→07 dots stay lit.
- **content-quality-checker**, 01 blocked: hatched and pulsing, and 10 cards
  read `⛔ via 01`.
