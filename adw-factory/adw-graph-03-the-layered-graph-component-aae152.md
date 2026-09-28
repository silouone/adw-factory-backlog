---
id: adw-graph-03-the-layered-graph-component-aae152
type: feat
status: in-progress
priority: 1
created: 2026-09-28
review: true
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [adw-graph-02-the-dependency-model-in-the-projection-e146f2]
attempts: []
---
# `<DepGraph/>`: the project's waves as columns, edges that never cross a card

> **Spec:** `specs/adw-v1.14-backlog-tab.md` §10, 2026-09-28, **G7** (and G1–G4
> as read from `BacklogProject.graph`, built by `adw-graph-02`).
>
> The operator picked prototype **variant A, the layered DAG**, over B (a wave
> list) and C (an ego graph):
> > "A is definitly the way"
>
> **Visual reference only:**
> `/Users/silouane/personal_project/adw-factory/.proto/backlog-graph/`
> (`bun .proto/backlog-graph/serve.ts`, then `http://127.0.0.1:4324/?proj=cmc&graph=cmc`).
> - It is throwaway, never committed, and absent from a run worktree.
> - The layout lives in `VariantA` in `entry.tsx`. Do not copy it; rebuild it
>   under Art. I.
>
> **Scope: the component only**, mounted by nothing yet.
> - The drawer and its triggers are `adw-graph-04`.
> - The live-state visuals are `adw-graph-06`.

## Requirements

- [ ] **R1 — red first, the layout is a pure helper.**
      `src/web/ui/graph-layout.ts` exports
      `layoutGraph(graph, {showDone}) → {nodes: {id,x,y,w,h}[], edges: {from,to,d}[], columns: {wave,x}[], width, height}`,
      where `d` is an SVG path.
      Tests go in `test/web/ui/graph-layout.test.ts` and are written first.
- [ ] **R2 — columns are waves.**
      - Wave 0 (the done / outside prerequisites) is the first column, shown
        only when `showDone`.
      - Column headers read `wave 1 · fires now`, `wave n · after wave n-1`,
        and `done / outside`.
      - Card size ≈ 212×50. The gap between columns ≈ 120, and the vertical
        gap between cards ≈ 22. The operator asked for the extra room, so
        keep it airy.
- [ ] **R3 — ordering.**
      - Barycenter sweeps order the nodes within each column: a few passes
        down, then up.
      - Each column is centred on the tallest one.
      - The output is deterministic: the same graph gives the same layout.
- [ ] **R4 — edge routing (the operator rejected lines drawn through cards).**
      - An edge spanning more than one wave gets a reserved pass-through slot
        (≈10 px) in every intermediate column. It runs horizontally through
        that slot.
      - The curves between columns are cubic, with horizontal tangents.
      - A card's incoming edges arrive spread down its left side, and its
        outgoing edges leave spread down its right side. Each set is ordered
        by the other end's y, so edges fan out instead of piling onto one
        point.
      - **The test that matters:** sample every edge path, 40 points per
        curve. On the adw-graph-02 cqc fixture and on a fixture with a
        3-wave-spanning edge, **no sample may fall strictly inside any card**.
- [ ] **R5 — `<DepGraph graph layout …/>`, a pure function of its props.**
      - **Cards** show: glyph · label · `P1` · status, and the title on a
        second line with an ellipsis. The left border is coloured by state,
        using the existing `s-<section>` palette. Manual and outside cards
        have dashed borders.
      - **Cards are opaque.** A card fades only its content, never its
        background, so an edge behind it never shows through.
      - **The critical path** (`edge.critical`, `node.critical`) is orange,
        `var(--model)`.
      - **Hover** lights the hovered node's full lineage: every ancestor and
        every descendant. Everything else dims to ≈0.22, edges included.
      - **Click** calls an `onSelect(id)` prop.
      - **The drift badge:** `⚠ n dep open` when `node.openDeps > 0`.
- [ ] **R6 — DOM tests** (happy-dom preload) cover:
      - one card per node;
      - a critical card and a critical edge carry their classes;
      - hover dims non-lineage cards;
      - click calls `onSelect`.
- [ ] **R7 — theme.** New rules live in the existing CSS module, scoped
      `.dg-*` (or similar), on the theme tokens (`--ok --warn --bad --live
      --model --line --panel --panel2`). Add no new colours beyond the
      prototype's red/blue alphas.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`. Visual check happens
when `adw-graph-04` mounts it.
