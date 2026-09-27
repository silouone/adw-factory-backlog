---
id: adw-fe-22-the-gantt-axis-is-not-time
type: bug
status: done
priority: 1
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-fe-22-the-gantt-axis-is-not-time-1789719265397","branch":"adw/adw-fe-22-the-gantt-axis-is-not-time","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-22-the-gantt-axis-is-not-time-1789719265397/workspace","outcome":"blocked","provider":"claude","model":"sonnet","pr":"https://github.com/silouone/adw-factory/pull/77"}]
---
# Seven instant steps worth 0.7% of a run consume 56% of the Gantt, and the four agent lanes that spent the money get 23%

> Observed 2026-09-18 by the operator on the run screen for
> `adw-bug-12-a-reviewer-formatting-slip-kills-a-green-run-and-leaves-no-evidence-1789684235106`:
> *"timings ARE NOT what they appear, plan/build/build-fix, which are the main
> STEPS, are way TOO small and we can't read the boxes inside. timeline is not
> linear which is kinda of a pb we introduced trying to fix it."*
>
> **Spec authority:** `specs/adw-v1.10-linear-timeline.md`, which amends
> `adw-v1.7-run-panel.md`'s "Solution" and "The warp". Read it first — this
> ticket deliberately reverses a decision v1.7 made on purpose, and the
> amendment is what makes that legal under `CLAUDE.md`'s amendment rule.

## Evidence

`src/web/timeline.ts`'s `layoutTimeline` gives every box narrower than
`MIN_READABLE_WIDTH_PCT = 8` a fixed 8% slice, then shares the *remainder*
among the rest. The total is pinned to 100% of the viewport, so every percent
handed to an instant node is taken from a working one.

Replayed from that run's own journal through the real `buildGanttView` +
`layoutTimeline` (span 57m 55s):

| box | lane | duration | real % | drawn % |
|---|---|---|---|---|
| `dispatch +1` | workspace | 688ms | 0.02 | **8.00** |
| `baseline` | workspace | 25m 51s | 44.62 | 21.05 |
| `baseline-green-check +1` | workspace | 2ms | 0.00 | **8.00** |
| `plan` | plan | 6m 25s | 11.07 | **5.22** |
| `assemble-test-only` | workspace | 1ms | 0.00 | **8.00** |
| `build-test-only` | build-test-only | 8m 7s | 14.02 | **6.61** |
| `red-check` | workspace | 2.6s | 0.07 | **8.00** |
| `assemble-fix` | workspace | 1ms | 0.00 | **8.00** |
| `build-fix` | build-fix | 13m 39s | 23.57 | **11.12** |
| `gates` | workspace | 3m 45s | 6.48 | 8.00 |
| `commit +2` | workspace | 4.8s | 0.14 | **8.00** |

Seven boxes worth **0.7% of the run's wall clock consume 56% of the canvas**.
The agent lanes — 48.7% of the elapsed time — are compressed to 23%. Across 28
replayed journals the pattern is consistent: `real 95% → drawn 44%`,
`93% → 44%`, `87% → 48%`.

**The screen contradicts itself.** `laneShare` (`render-run.ts`) prints
`share 51% / 11% / 14% / 24%` from **real** time, in the lane rail, one column
left of a chart drawing 45% / 5% / 7% / 11%.

## Root cause — one file, one function

`src/web/timeline.ts`, `layoutTimeline`, the block commented `--- warp ---`:

```ts
const smallCount = working.filter(isSmall).length;
const fixed = smallCount * MIN_READABLE_WIDTH_PCT;   // 7 * 8 = 56
const room  = Math.max(100 - fixed, 0);              // 44 left for real work
```

`MIN_READABLE_WIDTH_PCT` is `8`. `room` is what the boxes that actually took
time have to share. Nothing downstream is wrong: `buildGanttView` is correct,
`renderLanes` faithfully draws what it is handed.

## Requirements

Every item is a checkbox. All of them are pure-function behaviour on
`layoutTimeline` (`src/web/timeline.ts`) except R7.

- [ ] **R1 — linear.** A box's `left` and `width` are true percentages of the
      run's `[lo, hi]` span. Two boxes whose real durations are in a 3:1 ratio
      have widths in a 3:1 ratio. Real idle time between two boxes appears as
      a real gap. Boxes no longer sum to 100 and are no longer laid end to
      end — **the existing "no overlap, total exactly 100" expectations are
      wrong after this ticket and must be replaced, not bent.**
- [ ] **R2 — markers.** `DrawnBox` gains `render: "box" | "marker"`. A box is
      a `"marker"` iff its true width is `< MARKER_MAX_WIDTH_PCT` (`2`).
      `MIN_READABLE_WIDTH_PCT` is deleted; no box is ever given width it did
      not earn. **This decides the degenerate case, with no exception:** a run
      whose whole span is zero (one instant block, or several sharing one
      timestamp) has every box at `left: 0, width: 0`, so every one of them is
      a marker. It does **not** keep today's `left:0, width:100` — that value
      is an artefact of the warp's `scale = 100/rawTotal` normalisation and
      dies with it. `timeline.ts`'s existing `span > 0` guard in `pctOf` is
      already correct for this and needs no change.
- [ ] **R3 — a marker is still a box in the list.** Markers stay in
      `layout.boxes`, in execution order. Never filtered out — v1.7 user
      story 11 ("exactly one rail entry per box I can see on the Gantt")
      must still hold, and `renderRunPanel`'s rail must list markers with
      their real durations.
- [ ] **R4 — markers cluster, boxes do not.** A maximal run of consecutive,
      **same-lane** markers with no box between them collapses into one
      marker, labelled `"<first> +N"`. The rule is gap-independent: the old
      gap-aware clustering existed only because small boxes were being
      widened. Two boxes never merge.
- [ ] **R5 — the gate exemption moves from kind to form.** A gate-kind block
      drawn as a **box** still never absorbs a neighbour into a cluster and
      still renders its full verdict strip (the `adw-fe-19` guarantee,
      preserved). A gate-kind block below the marker threshold **is** a
      marker like any other — 10 genuinely instant gate blocks exist in the
      banked runs (`baseline` at 0.00%) — and carries its worst-of-members
      outcome colour, so a failed instant gate is still a red mark.
- [ ] **R6 — no two boxes in one lane overlap.** For every lane, sorting its
      drawn boxes by `left`, each box's `left + width` is `<=` the next
      box's `left`. This is `adw-fe-19` defect A, restated for the new
      geometry, and R4 is what makes it true for markers.
- [ ] **R7 — the axis stops lying in prose too.** `toWarped` is deleted from
      `TimelineLayout` (verified: **no consumer outside `timeline.ts` and its
      own tests** — the SSE path in `server.ts` does not read it). `ticks` are
      `ticksFor(span)` unmapped, so they are evenly spaced again. The
      `.ax-note` string `non-linear — instant steps widened to stay readable`
      is removed from `renderLanes` (`render-run.ts`).
- [ ] **R8 — markers render as marks.** `renderDrawnBox` gives a
      `render: "marker"` box its own class and a fixed, small footprint
      centred on its `left`, carrying its outcome glyph and colour only — no
      name, no duration, no purpose line, no gate strip inside it (all of
      which are in the rail and panel). A `"box"` renders exactly as it does
      today.
- [ ] **R9 — retire `.nd { min-width: 26px }`.** That floor (`render-run.ts`,
      the `.nd` rule) is the ORIGINAL `adw-fe-19` rescue mechanism, and
      markers are what replace it. Left in place it silently reintroduces the
      very bug this ticket closes: a box just above the marker threshold is
      inflated by CSS past its true right edge and into its neighbour. The
      live band is real — at the `@media(max-width:900px)` layout the file
      already supports, the track is ~760px and 26px is **3.4%**, so any box
      between 2% and 3.4% is drawn wide and then pushed wider. Two such boxes
      exist in the banked runs (`review-standards` at 2.33% — a node that
      spent tokens — and `baseline` at 3.18%).
      **R6 cannot catch this**: R6 asserts on layout percentages, and the
      inflation happens in CSS afterwards. It has to be removed, not tested
      around. The accepted consequence: a box just over the threshold renders
      as a thin sliver at `fit` zoom. That is honest, and `adw-fe-23`'s zoom
      is its remedy.

## Build protocol

TDD, Article I — **no source before a reviewed, red test.** This is a `bug`
ticket, so `base-green-check` and `red-check` bracket the work; write the
failing test first and let the machine watch it go red.

1. **Red tests** in `test/web/timeline.test.ts` (the pure seam — no DOM, per
   Art. III), plus the two rendering assertions in
   `test/web/render-run.test.ts`. The sharpest ones, in order:

   - **R1, the reported defect, from the reported numbers.** Build the
     fixture from the evidence table above (7 instant + 4 real nodes across 4
     lanes, a 57m 55s span) and assert `plan`, `build-test-only` and
     `build-fix` are laid out within 0.05 of `11.07`, `14.02` and `23.57` —
     not `5.22`, `6.61`, `11.12`. One test that fails today for exactly the
     reason the operator reported.
   - **R1, ratio.** Two large blocks 3:1 in real duration keep 3:1 in width
     *and* the smaller one is no longer inflated by a sibling's presence:
     adding a third, instant block to the fixture must not change either
     width.
   - **R2/R3.** An instant block yields `render: "marker"` **and** still
     appears in `layout.boxes`; a long one yields `render: "box"`.
   - **R4.** Three consecutive same-lane instant blocks collapse to one
     marker carrying all three members; the same three split by a real box
     in the middle yield two markers.
   - **R5.** A gate block above the threshold stays a box, never merges, and
     still renders its verdict chips; a 0ms gate block yields a marker
     carrying its outcome class.
   - **R6.** For the evidence fixture, per lane, no box's right edge passes
     the next box's left edge — `adw-fe-19`'s regression, restated.
   - **R7.** `layout.ticks` are evenly spaced (equal deltas) for a span with
     instant blocks in it; `renderLanes`'s HTML no longer contains
     `non-linear`.

2. **Confirm red.** Present for operator review before any source.
3. Implement to green.
4. `bun run lint && bunx tsc --noEmit && bun test` — all green.

## Blast radius — read before estimating

- `src/web/timeline.ts` — `layoutTimeline`'s warp and clustering blocks,
  `MIN_READABLE_WIDTH_PCT`, `toWarped`, `TimelineLayout`, `DrawnBox`,
  `WorkingBox.mergeable`. The file header comment documents the warp at
  length and must be rewritten, not left contradicting the code.
- `src/web/render-run.ts` — `renderDrawnBox` (marker branch), `renderLanes`
  (the `.ax-note`), and the `.nd` CSS family (a `.nd-marker` rule). The
  header comment at `:185-191` also describes the warp.
- `src/web/server.ts` — **no change expected**; it calls `renderLanes` and
  `renderRunPanel`, neither of whose signatures move. Confirm it still
  compiles rather than assuming.
- `test/web/timeline.test.ts` — the `describe("proportion")` and
  `describe("no overlap, total exactly 100")` blocks assert the warp
  directly and are replaced by R1/R6. `describe("clustering")`'s
  `MIN_READABLE_WIDTH_PCT` boundary tests (the "EXACTLY" pair) and
  `describe("tick alignment")`'s four `toWarped` tests test machinery that
  ceases to exist — delete them with the machinery, do not port them.
  `describe("degenerate single-instant span")`'s two tests assert
  `left:0, width:100` for a zero-span run (`:496`, `:519`). Per R2 that is
  now `left:0, width:0, render:"marker"` — **update these expectations, and
  expect them red before you do.** They are not a regression; they encode the
  warp's normalisation.
- `test/web/render-run.test.ts` — the `expect(baseline.width).toBeCloseTo(8, 0)`
  assertion at `:1485` and the comment block at `:1420-1432` encode the
  warp's guarantee. `blocksOf` (`:1436`) parses `class="nd ..."` and will
  not match a marker's class — extend it or it silently returns fewer boxes
  and tests pass vacuously.

## Out of scope

- The zoom control and the horizontal scroll container —
  `adw-fe-23-zoom-and-scroll-the-run-timeline`, which depends on this.
- Any change to `src/web/gantt.ts`. `buildGanttView`'s pairing, lane
  assignment and `GanttView` shape are correct and untouched.
- The rail's ordering and content, the panel, the metric tri-state (v1.7,
  all unchanged).

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
- [ ] `bun test test/web/timeline.test.ts test/web/render-run.test.ts` green.
- [ ] Open the run screen for
      `adw-bug-12-…-1789684235106` and read the `share` meter against the
      chart: the lane rail's `51% / 11% / 14% / 24%` and the drawn widths now
      agree.
