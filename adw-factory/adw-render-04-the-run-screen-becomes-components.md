---
id: adw-render-04-the-run-screen-becomes-components
type: feat
status: done
priority: 2
created: 2026-09-18
caps: {minutes: 180, turns: 900}
depends: [adw-render-03-the-board-becomes-components]
attempts: [{"runId":"adw-render-04-the-run-screen-becomes-components-1789903616553","branch":"adw/adw-render-04-the-run-screen-becomes-components","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-render-04-the-run-screen-becomes-components-1789903616553/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/99","provider":"claude","model":"sonnet"}]
---
# P3 — the run screen becomes components, and the reader stops scrolling itself to the top

> **Spec authority:** `specs/adw-v1.11-render-architecture.md` §4 **P3**.
> **Depends on P2** rather than running beside it: the usage chip and panel
> are one shared component across both screens ("same component, same data,
> same gesture" — `adw-usage-03`), and converting it twice in parallel would
> produce two of it.

## The defect being retired

Measured 2026-09-18 on the run screen: **11 pushes in 10 s against 0 new
journal bytes**, each replacing both `#run-console` and `#dock` wholesale.
Confirmed lost every tick: the panel reader's `scrollTop` (1400 → **0**,
unconditionally — a replaced element's scroll is always 0), a 5,825-character
text selection, keyboard focus, and the md/raw render memo
(`md.dataset.done`), which is rebuilt from scratch every second.

The control proves the mechanism: the **same page for a finished run** pushes
**1 frame in 12 s** and the reader's scroll **holds**.

## Requirements

- [ ] **R1 — `render-run.ts` is replaced by components.** Retired:
      `renderRunPage`, `renderRunHeader`, `renderLanes`, `renderUpNext`,
      `renderRunPanel`'s HTML emission, the client IIFE, `showScope`,
      `closeDock`, `applyMd`, and **`DOCK_SPLIT_MARKER`** — the two-region
      split exists only because one string had to carry two `innerHTML`
      targets.
- [ ] **R2 — dock state is component state.** The active scope, the md/raw
      toggle, and the full-screen flag become signals. `showScope(active)`'s
      restore-after-swap call in `es.onmessage` is **deleted**, not ported.
- [ ] **R3 — the md render stops being a manual memo.** `md.dataset.done`
      exists to avoid re-parsing markdown on every swap. With no swap, render
      markdown normally.
- [ ] **R4 — `adw-fe-24`'s prompts stay visible.** fe-24 restored the
      compiled prompts to the live path and P1 carried them into the
      view-model. Assert a component test that the panel shows prompt text
      **after** a view-model update, not just on first paint. This is the
      regression that started the whole investigation.
- [ ] **R5 — no new derivation.** `gantt.ts`, `timeline.ts`, `drawer.ts`,
      `run-view.ts`, `capture.ts` and `usage.ts` are **not modified**.
      `timeline.ts` in particular will by then carry `adw-fe-22`'s linear
      axis — consume it, do not re-derive it.
- [ ] **R6 — v1.7 and v1.10's view contracts hold.** The panel, the rail, the
      one-entry-per-box rule, the metric tri-state and the linear axis are
      preserved verbatim (§"Does NOT amend"). This ticket changes how a view
      is expressed, never what it says.

## Verify

- [ ] Red test first (Art. I): the panel reader's scroll position survives a
      view-model update. RED today (1400 → 0).
- [ ] Red test: the active dock scope and md/raw toggle survive an update.
- [ ] Red test for R4: prompt text is present after a simulated live update.
- [ ] Manual, against a live run: scroll the reader, select text, switch to
      md, open a node — all survive ≥10 ticks.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Unblocks

`adw-fe-23-zoom-and-scroll-the-run-timeline` was set `status: blocked` on
2026-09-18 against this ticket. Its **R4** (capture `scrollLeft` and the zoom
class before the `innerHTML` assignment, restore after — the band-aid a fifth
time) becomes **free** here: zoom and scroll are component state. Re-queue
fe-23 in P4 and strike its R4.

## Out of scope

- The zoom control itself — that is fe-23, after this lands.
- Redesign (§8).
