---
id: adw-fe-13-run-screen-design-rework
type: feat
status: done
priority: 1
created: 2026-09-14
depends: [adw-fe-12-wire-the-run-screen]
attempts: [{"runId":"adw-fe-13-run-screen-design-rework-1789382128624","branch":"adw/adw-fe-13-run-screen-design-rework","workspace":"/Users/silouane/adw-factory/runs/adw-fe-13-run-screen-design-rework-1789382128624/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-13-run-screen-design-rework-1789391893971","branch":"adw/adw-fe-13-run-screen-design-rework-2","workspace":"/Users/silouane/adw-factory/runs/adw-fe-13-run-screen-design-rework-1789391893971/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/39","provider":"claude","model":"sonnet"}]
---
# The run screen, rebuilt to the Console design — and one CSS bug that made it unreadable

> **Design is settled.** Operator picked variant **D — Console** from the
> prototype on 2026-09-14 ("that's good enough for a v1"). Build to it.
>
> **Reference implementation:** `src/web/render-run.prototype.ts`, variant D
> (~276 lines) + `src/web/PROTOTYPE-NOTES.md`. It runs: `just web`, open a run,
> append `&variant=D`.
>
> **NO SPEC AMENDMENT NEEDED — and that is a result, not an omission.** v1.2
> decision 11 says *"zero new dependencies… two screens of positioned
> rectangles and a drawer do not need a framework."* The prototype is **vanilla
> CSS + ~25 lines of JS** and clears the bar, so it answers
> `specs/adw-v1.5-operator-console.md` **D1** ("does the factory take a frontend
> dependency?") with **no**. v1.5 stays blocked for its OTHER asks — filters,
> dependency view — not for this.

## 1. The bug this also fixes

`src/web/render-run.ts` computes correct `left`/`width` percentages from real
timestamps, then applies them to:

```css
.block { position: relative; display: inline-block; ... }
```

Percentage offsets need `position: absolute`. As written, blocks flow inline
**and then** shift, so they pile up (`dispa provis basel greer` overlapping in
the operator's screenshot) and a `Math.max(widthPct, 1)` floor clips the labels
of sub-second nodes.

Fix: `position: absolute`, an explicit height on the track, and a narrow block
**collapses to a marker** rather than clipping — see §4.

## 2. Structure

```
┌ crumb: ← all runs
├ HEADER   [status pill] ticket-id …………… [dur][turns][billed][context] chips
├ AXIS     ⌐ rail gutter ⌐  0s    15m    30m    45m    60m
├ LANES    one row per lane; vertical gridlines continue through every row
│   rail:  ⌘/◆ role · model-or-"deterministic · no tokens" · share meter
│   track: absolutely-positioned blocks
└ DOCK     sticky bottom panel; click a block to fill it, ✕ or Esc to close
```

**Lanes come from `buildGanttView` unchanged.** `gantt.ts` already puts every
deterministic and gate node in the single `workspace` lane and gives each agent
node its own — do not re-derive that.

## 3. The three block kinds must look different

This is the ticket's core requirement, not decoration. `kindOf()` in the
prototype splits them; the distinction is Art. III made visible — *"never an
agent where a function suffices"* is invisible if a free node and a
token-burning one look the same.

| kind | how to tell | treatment | why |
|---|---|---|---|
| **agent** | has `metrics`/`usage` | solid, lane-hued, purpose line, tool-call ticks | spends tokens |
| **gate** | has `gates[]` | slate surface, **gate verdicts inline** | spends wall clock |
| **deterministic** | neither | dashed hairline, dimmed, no fill | **free** |

- [ ] **A gate block renders its verdicts inline** — `✓ lint ✓ typecheck ✕ test`
      — never behind a click. Measured 2026-09-14: the gate suite is ~50% of a
      run's wall clock, and "did the test gate go red" is the single thing an
      operator most often opens a run to find out. On the bug lane this makes
      `red-check`'s deliberate `✕ test` readable from the chart — the factory's
      central claim, visible.
- [ ] The lane rail states what the lane IS: `⌘ workspace / deterministic · no
      tokens` vs `◆ plan / claude-sonnet-5`.

## 4. Blocks are cards, and narrow blocks are markers

- [ ] A block shows `✓ name · duration`, then **what the node is for** —
      "Turn the ticket into an implementable plan". A block that says what it
      does beats one that only says what it is called. The node set is fixed
      and small, so this is a lookup (`PURPOSE` in the prototype), not a guess;
      `assemble-*` falls back by prefix.
- [ ] Tool calls render as **ticks along the block's foot**, positioned by
      their own timestamps from `adw-fe-07`'s capture dots.
- [ ] A block too narrow for a card (<~7% of the span) collapses to a
      **marker** — outcome glyph + the node's initial — and keeps its full name
      in `title`. Truncation is never silent.

## 5. Visual language

Palette — carry these verbatim; they are chosen, not defaults:

```
--bg    #0A0B0F   --panel #10131A   --panel2 #141822   --line #1F2532
--ink   #E4E8F1   --dim   #727C90   --faint  #3A4254
--model #E0873C   --tool  #3F87A6          <- the thesis, below
--ok    #5BC08A   --bad   #E0615C   --warn  #E0B44A   --live #7CC5FF
```

- [ ] **Lane hue is deterministic by index** (`HUES` in the prototype), so a
      role is recognisable before it is read and no palette lookup can drift.
- [ ] **model vs tool colour is the page's thesis.** This session measured
      agents at ~90% model time; that is the run's most important fact and the
      old screen buried it in a text line. Burnt orange = model (thinking),
      slate blue = tool (waiting on the machine). Used in the dock's split bar
      and anywhere the split appears.
- [ ] Type: `ui-monospace` throughout. It is a control plane, not a document.
- [ ] Quality floor, unannounced: responsive to ~900px, visible keyboard focus,
      `prefers-reduced-motion` respected.

## 6. Two things the design deliberately REFUSES

Both were in the operator's reference design. Do not add them back without a
decision.

- [ ] **No cost chip.** Nothing in the factory records a price. A `$` that is
      really a token count would be the most confident lie on the screen.
- [ ] **No CONTEXT % meter.** `metrics.ts` is explicit that `contextTokens` is
      *"an absolute token count rather than a percentage of a guessed
      context-window denominator"* — the repo refused that guess once already.
      The lane meter is **share of the run's wall clock**: derived, true, and
      the number this session kept rediscovering.

## 7. Architecture — unchanged from v1.2

- [ ] Renderers stay **pure**: a view object in, a string out, no I/O. Spec
      decision 2 — *"the pure/edge split is mandatory, not stylistic"* — and it
      is what lets these be unit-tested against hand-built view objects.
- [ ] `server.ts` stays a thin shell, **read-only**. No mutating route.
- [ ] **Zero new runtime dependencies** (decision 11). Hand-written CSS and
      vanilla JS only. The prototype proves the ceiling is high enough.
- [ ] The live `/events` SSE path (`adw-fe-08`) must keep working: the grid's
      three render states (`running` / `finished` / `unknown`) are untouched by
      this ticket, and a live run's blocks must still update in place.

## Amendment 2026-09-18 (adw-fe-15) — the figure, re-derived

§5's thesis claim — *"This session measured agents at ~90% model time"* — was
recorded before `adw-bug-01` was found. `adw-bug-01` §2 (Defect B) established
that, at the time, `metrics.ts` was reporting **100% model time, 0% tool
time, on every node of every run** — a fabricated split, not an absence. That
instrument could not have measured "~90%" of anything real.

**The re-derived figure is 74.6% model time.** Measured 2026-09-18 by
`just model-tool-split` (`scripts/run-model-tool-split.ts`) against the
operator's banked journals, post-`adw-bug-01`:

| | |
|---|---|
| runs read | 43 |
| blocks carrying a resolved time split | 105 (agent blocks only — `metrics.ts` attaches no split to a deterministic or gate block) |
| model time | 965.9m |
| tool time | 328.0m |
| blocked-on-subagent | 0.0m |
| **model-time share** | **74.6%** |

**The thesis holds; the number did not.** Model time still dominates by
roughly 3:1, so §5's design conclusion — that the model-vs-tool colour split
is worth carrying on the page — survives re-derivation. But "~90%" overstated
it by some 15 points, and tool time is a quarter of agent wall-clock rather
than the rounding error the original figure implied. Any copy, legend or
colour-weighting tuned to "~90%" should be re-read against 74.6%.

**Reproducing it.** `runs/` is gitignored, so this figure is not reproducible
from a clone — the journals live only on the disk that produced them. Quote
`just model-tool-split`, never this remembered number; the same rule the
README already states for `bun scripts/run-metrics.ts`.

## Verify

- [ ] A `feat` run and a **`bug` run** both render legibly — the bug lane is
      the hard case (six roles, `red-check`'s deliberate red).
- [ ] No block overlaps another; every block's left edge matches its real start
      time against the axis.
- [ ] A sub-second node is a marker with its name in `title`, never clipped text.
- [ ] A gate block shows each gate's verdict without a click; a red gate is
      unmistakably red.
- [ ] Renderers are unit-tested against hand-built view objects, no server.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## When done

- [ ] **Delete** `src/web/render-run.prototype.ts`, `src/web/PROTOTYPE-NOTES.md`
      and the `?variant=` branch in `server.ts`.
- [ ] **Do not promote prototype code.** It was written under prototype rules —
      no tests, minimal error handling. Rewrite it properly into
      `render-run.ts`.

## Out of scope

The grid screen (still cards, by design), per-target filtering, the dependency
view, and host RAM/CPU — all `adw-console-01`, blocked on the operator.
