---
id: adw-render-03-the-board-becomes-components
type: feat
status: done
priority: 2
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: [adw-render-02-the-wire-carries-view-models]
attempts: [{"runId":"adw-render-03-the-board-becomes-components-1789846487294","branch":"adw/adw-render-03-the-board-becomes-components","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-render-03-the-board-becomes-components-1789846487294/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# P2 — the board becomes components, and a collapsed section stays collapsed

> **Spec authority:** `specs/adw-v1.11-render-architecture.md` §4 **P2**.
> Follow the pattern `adw-render-01-foundation` established; do not invent a
> second one.

## The defect being retired

Measured 2026-09-18 on the board: every SSE frame replaces **5,158 DOM nodes**
from a **302.6 KB** string via `#grid-body.innerHTML`. Confirmed lost on every
swap: `<details>` expansion (collapsed → re-opened within 1.1 s), window
scroll (838 → 0), text selection (143 chars → 0) and keyboard focus (a focused
`<a>` → `document.body`).

The closure-hoisting mitigation has been applied twice on this screen already
(`adw-usage-01` PR #61 for the project dropdown; `adw-usage-02` for the usage
chip). Both are **deleted** by this ticket — the state they hand-restore
becomes ordinary component state.

## Requirements

- [ ] **R1 — `render.ts` is replaced by components** under `src/web/ui/`.
      Retired: `renderGridBody`, `renderGrid`, `renderBoardHeader`,
      `filterScript`, and the `#grid-body.innerHTML` assignment.
- [ ] **R2 — filter state is component state.** `proj`, `health` and the
      dropdown's open flag live in signals. The hand-rolled
      `apply()`-after-every-swap restore and the `menuOpen` capture in
      `es.onmessage` are **deleted**, not ported.
- [ ] **R3 — the usage chip/panel keep their current behaviour.** They were
      deliberately mounted outside the swapped fragment (D5 of
      `adw-usage-02`); as components that special case disappears, but the
      60 s TTL refresh and the pinned-open behaviour must be preserved.
- [ ] **R4 — `queue.prototype.ts` and the `?queue=` branch are deleted.** The
      file's own header marks it THROWAWAY; it renders HTML strings and has
      no future. Its findings are already captured in
      `src/web/QUEUE-PROTOTYPE-NOTES.md` — delete that too, or fold it into
      `adw-console-01`.
- [ ] **R5 — no new derivation.** Components consume `RunView`/`CardSummary`
      from P1's payload. `board.ts`, `card.ts`, `projection.ts` and
      `metrics.ts` are **not modified** — if a component seems to need new
      derivation, it belongs in those modules with its own test.
- [ ] **R6 — the board looks identical.** §8: no redesign. The CSS ported in
      P0 applies unchanged.

## Verify

- [ ] Red test first (Art. I): a component test asserting a collapsed section
      stays collapsed across a view-model update. RED today.
- [ ] Red test: filter selection survives a view-model update.
- [ ] Manual, against a live run with the clock ticking: collapse a section,
      scroll down, select text, focus a card link — all four survive ≥10
      ticks. (Spike baseline 2026-09-18: **0 childList mutations** on the
      section across 9 clock ticks.)
- [ ] `grep -r "innerHTML" src/web/` returns nothing for the board path.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

- The run screen (P3).
- Deleting the retired tests — P4 sweeps. This ticket may leave
  `render.test.ts` failing-and-skipped **only if** it says so explicitly in
  the PR body; preferably rewrite its assertions as component tests here.
