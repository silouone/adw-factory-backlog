---
id: adw-graph-04-the-graph-drawer-661279
type: feat
status: in-progress
priority: 1
created: 2026-09-28
review: true
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [adw-graph-03-the-layered-graph-component-aae152]
attempts: [{"runId":"adw-graph-04-the-graph-drawer-661279-1790636751031","branch":"adw/adw-graph-04-the-graph-drawer-661279","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-graph-04-the-graph-drawer-661279-1790636751031/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/157","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The backlog screen's graph drawer: docked, zoomable, one click from any dependency

> **Spec:** `specs/adw-v1.14-backlog-tab.md` §10, 2026-09-28, **G8**.
>
> The operator on the round-1 prototype:
> > "we need a real drawer though not a mixed drawer vs floating modal"
>
> The ticket panel was also a floating card, so **both** dock.
>
> **Visual reference only:**
> `/Users/silouane/personal_project/adw-factory/.proto/backlog-graph/`
> (`entry.tsx` `GraphDrawer`/`Canvas`, `state.ts`, `graph.css`, and the
> `fork/` copy of `Backlog.tsx`). It is throwaway and absent from a run
> worktree.
>
> **Dock default: bottom.** The operator saw both docks but did not pick one.
> Change the default here before firing if they prefer right.

## Requirements

- [ ] **R1 — red first.** The drawer's derivations are pure helpers, tested
      before any component:
      - **size resolution:** preset → px, clamped to the room left of the
        panel;
      - **push padding:** the table's `padding-right` / `padding-bottom`;
      - **zoom math:** clamp to 20–250%; zooming about an anchor keeps that
        viewport point fixed; fit = `min(viewW/w, viewH/h, 1)`, and a
        zero-size viewport is a no-op;
      - **the URL keys.**
- [ ] **R2 — a real drawer.**
      - **Bottom** (default): full width left of the panel, heights 30% /
        45% / full, and a drag handle on the top edge.
      - **Right**: left of the panel, widths 480 px / 50 vw / full, and a drag
        handle on the left edge.
      - Both are flush to the edge: no radius, no shadow, no gap.
      - A `⬓ bottom | ◨ right` toggle sits in the drawer header.
- [ ] **R3 — the ticket panel docks too.**
      - `.bl-panel` becomes flush right and full height, with a left border
        only.
      - When both are open, the panel is on the edge and the graph sits to
        its left (right dock) or ends at the panel's edge (bottom dock).
      - **The table is pushed, never covered.** This replaces today's fixed
        `.bl-has-panel{padding-right:690px}`.
- [ ] **R4 — triggers.**
      - An `⎇ graph` button at the right end of the project chip strip.
      - The `g` key toggles the drawer.
      - Any `[data-dep-id]` chip (the panel's dep chips, and `adw-graph-05`'s
        why-cell chips) opens the graph on that dep's project, focused on
        that dep.
      - With "All" selected, the drawer opens on the filtered project, else
        the first project with deps. Its header carries project chips.
- [ ] **R5 — selection.**
      - Clicking a card selects that row (`selectedRowId`), so its panel
        opens beside the drawer.
      - While the drawer is open, it follows the selection: select a row,
        and the graph refocuses on it.
      - `esc` still closes the panel first.
- [ ] **R6 — zoom and pan.**
      - Pinch or ⌘/ctrl-wheel zooms around the cursor. A plain wheel
        scrolls.
      - Dragging the empty canvas pans, with a grab cursor. Dragging on a
        card is a click.
      - The `−` · `%` (100%) · `+` · `fit` buttons, and the keys `-` `0` `+`
        `f`, work only while the drawer is open and focus is not in an input.
      - Switching project fits the graph, never above 100%.
- [ ] **R7 — `show done` toggle** (maps to `layoutGraph({showDone})`), and a
      legend.
- [ ] **R8 — URL state**, merged next to the backlog's own keys:
      `graph` (project), `focus`, `dock`, `gw`/`gh` (size). A reload restores
      the drawer.
- [ ] **R9 — DOM tests:**
      - the button and `g` open and close the drawer;
      - a `[data-dep-id]` click opens it focused;
      - a card click selects the row;
      - the table padding matches the docked sizes;
      - the dock toggle moves it.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` →
Backlog, `cmc`:
1. `⎇ graph` opens a bottom drawer, and the table above it keeps full width
   and scrolls clear of it.
2. Click a card: the panel docks right, and the drawer stops at its edge.
3. Pinch-zoom around a card, drag to pan, then `f` fits.
4. Flip to `◨ right`. Reload: the same state comes back.
