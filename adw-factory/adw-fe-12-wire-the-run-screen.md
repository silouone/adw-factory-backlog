---
id: adw-fe-12-wire-the-run-screen
type: feat
status: done
priority: 1
created: 2026-09-14
depends: [adw-fe-06-gantt, adw-fe-07-tool-dots, adw-fe-09-node-drawer, adw-fe-10-metrics-rail]
attempts: [{"runId":"adw-fe-12-wire-the-run-screen-1789371966724","branch":"adw/adw-fe-12-wire-the-run-screen-2","workspace":"/Users/silouane/adw-factory/runs/adw-fe-12-wire-the-run-screen-1789371966724/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/36","provider":"claude","model":"sonnet"}]
---
# 946 lines of finished view logic are unreachable — `adw web` serves one route

> Minted 2026-09-14 from the operator's own screenshot of `adw web`: a wall of
> grid cards, and nothing else reachable. Part of the v1.2 live view
> (`specs/adw-v1.2-live-view.md`) — **no spec change**, this only connects what
> that amendment already specified and four merged tickets already built.

## Evidence

```
src/web/server.ts:66
  if (req.method !== "GET" || url.pathname !== "/") {
```

One route. Meanwhile, merged, tested and **unreachable**:

| module | lines | built by |
|---|---|---|
| `src/web/gantt.ts` | 240 | `adw-fe-06-gantt` |
| `src/web/capture.ts` | 282 | `adw-fe-07-tool-dots` |
| `src/web/drawer.ts` | 228 | `adw-fe-09-node-drawer` |
| `src/web/metrics.ts` | 196 | `adw-fe-10-metrics-rail` |
| | **946** | |

Each of those tickets was scoped *"projection-only, no server or route wiring"*
— deliberately, so they could be built in parallel without colliding in
`server.ts`. That worked. What no ticket then did is **compose them**. `fe-08`
is the SSE tailer and `fe-11` is an operator-executed acceptance bar; neither
adds a route.

The spec promised **two screens**:

> 1. The run grid — one card per run… 2. The run Gantt — roles as horizontal
> rows, time as the x-axis… Selecting a block opens a drawer with that node's
> owner, kind, attempt, gates, and its compiled prompt.

Screen 1 ships. Screen 2 exists as data and cannot be looked at.

## The part that is NOT just a route

The four modules export **projections and view types only** — `buildGanttView`,
`attributeDots`, `attributeMetrics`, and the `DrawerView` / `NodeMetrics` /
`GanttView` shapes. **None of them exports a renderer.** `render.ts` has exactly
one, `renderGrid`.

So this ticket is *render + route*, not *route*. Scope it honestly: the
projections are done and carry every rendering rule; what is missing is the
HTML for them and the server shell that serves it.

## Requirements

- [ ] A per-run screen is reachable — the Gantt, with tool-call dots inside the
      blocks, the metrics rail, and the drawer on selecting a block.
- [ ] A grid card links to its run's screen. Today a card is a dead end.
- [ ] The renderers stay in the **pure** half: a view object in, a string out,
      no I/O. Spec decision 2 — *"the pure/edge split is mandatory, not
      stylistic"* — and it is what lets these be tested with plain arrays.
- [ ] `server.ts` stays a thin shell with injected dependencies, and remains
      **read-only**: no mutating route exists, and none is added here.
- [ ] **Zero new runtime dependencies** (spec decision 11). `Bun.serve` and
      hand-written HTML only.
- [ ] A run whose capture is missing must render differently from one captured
      with zero tool calls — `adw-fe-07` already draws that distinction in the
      data and it must survive into the pixels.

## Verify

- `adw web`, open a run from the grid, see its Gantt with dots, rail and drawer.
- A run with a missing journal still renders the grid (the existing skip line).
- Renderers are unit-tested against hand-built view objects, no server.
- `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Live updates (`adw-fe-08-live-sse`), and anything the operator asked for that
v1.2 never scoped — per-target filtering, a dependency view, host RAM/CPU, or a
frontend framework. Those are `specs/adw-v1.5-operator-console.md`, proposed
separately and **not approved**.
