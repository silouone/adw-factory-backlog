# Amendment v1.10 — the timeline tells the truth about time

> **Status:** proposed 2026-09-18, from operator review of the run screen on
> `adw-bug-12-…-1789684235106`, with the geometry replayed from 28 banked
> journals through the real `buildGanttView` + `layoutTimeline`.
> **Amends:** `adw-v1.7-run-panel.md` — its "Solution" section, the paragraph
> beginning *"The time axis becomes piecewise-linear"*, and the
> "Implementation Decisions — The warp" section. Everything else v1.7
> specifies (the panel, the rail, the one-entry-per-box contract, the metric
> tri-state) is **unchanged and load-bearing here**.
> **Binding:** `constitution.md` — unchanged.
> **Implemented by:** `tickets/adw-fe-22-the-gantt-axis-is-not-time.md`
> (the layout) and `tickets/adw-fe-23-zoom-and-scroll-the-run-timeline.md`
> (the container).

## Problem Statement

v1.7 fixed a real bug — instant nodes rendered as zero-width glyphs or
overlapping 26px boxes (`adw-fe-19` defect A) — by giving every sub-threshold
box a fixed `MIN_READABLE_WIDTH_PCT = 8` slice and sharing the *remainder*
among the rest.

The total is pinned to 100% of the viewport, so **the warp is a zero-sum
transfer**: every percent handed to an instant node is taken from a working
one. Replayed from the run above (57m 55s):

| box | duration | real % | drawn % |
|---|---|---|---|
| `baseline` | 25m 51s | 44.62 | 21.05 |
| `plan` | 6m 25s | 11.07 | **5.22** |
| `build-test-only` | 8m 7s | 14.02 | **6.61** |
| `build-fix` | 13m 39s | 23.57 | **11.12** |
| 7 instant boxes | **0.7% of the run, combined** | ~0 | **56.00** |

**Seven boxes worth 0.7% of the wall clock consume 56% of the canvas.** The
four agent lanes — 48.7% of the elapsed time, and 100% of the money — are
compressed into 23% of it. Across all 28 replayed runs the pattern holds:
agent lanes land at `real 95% → drawn 44%`, `93% → 44%`, `87% → 48%`.

Two operator-visible symptoms, one cause:

1. **The main steps are unreadably small** — *"plan/build/build-fix, which are
   the main STEPS, are way TOO small and we can't read the boxes inside."*
   They are small **because** the instant ones were widened.
2. **The axis is not linear** — *"timeline is not linear which is kinda of a
   pb we introduced trying to fix it."* The non-linearity **is** the widening
   mechanism, and the screen announces it in small grey type: `non-linear —
   instant steps widened to stay readable`.

**The screen already contradicts itself.** `laneShare` prints `share 51% /
11% / 14% / 24%` in the lane rail, computed from **real** time, one column
left of a chart drawing those same lanes at 45% / 5% / 7% / 11%.

**And v1.7 overshot its own source.** `adw-fe-19`, the ticket that raised the
defect, asked for something else entirely:

> An instant node must be visually distinct from a zero-length one — **a tick
> or marker on the lane, not a 26px box pretending to have duration.**

fe-19 asked for markers. v1.7 answered with the warp, and paid half the canvas
for it.

## Solution

**The axis is linear. Instant nodes are markers, not boxes. Zoom and
horizontal scroll are the readability valve.**

1. **Linear time.** A box's `left` and `width` are true percentages of the
   run's own span. A step that took twice as long is twice as wide, always.
   Gaps between boxes are real idle time and are drawn as such. The
   `non-linear` axis note is deleted because it stops being true.
2. **Markers below the floor.** A box whose true width falls under a small
   threshold is not drawn as a box at all — it is a marker at its true
   position, carrying its outcome colour. It is never given width it did not
   earn. It **remains a `DrawnBox` in `layout.boxes`**, so v1.7's user story
   11 ("exactly one rail entry per box I can see on the Gantt") holds
   unchanged, and the step's name, duration and detail stay one click away in
   the rail and panel v1.7 already built.
3. **Zoom + horizontal scroll.** The chart defaults to `fit` — the whole run
   on one screen, honestly proportioned. A zoom control widens the track
   beyond the viewport and the chart scrolls horizontally.

### Why scroll alone was not the answer

The operator's proposal was *"restore timing to what they really are, and
ENABLE horizontal scroll if it does not fit."* The first half is exactly
right. The second half cannot carry the load by itself: the run's dynamic
range is `25m 51s : 1ms` ≈ **1.55 × 10⁶ : 1**, so giving the 1ms node a 26px
floor under a linear scale needs a ~40-billion-pixel track. No zoom reaches
that.

What scroll actually buys is the removal of the **zero-sum constraint**: once
the total is not pinned to the viewport, widening stops being theft. But
instant nodes still need a different *form*, because width is never
affordable for them at any zoom. Markers are that form.

### Why `fit` is the default, and zoom is a control rather than a policy

Measured over the 28 journals, auto-zooming until every box shows its full
label needs a **median 2.6× at a 900px track** (p75 3.6×), which would put 21
of 26 runs behind a scrollbar on open and lose the whole-run shape the current
chart does at least preserve.

It is not needed, because linear + markers does most of the work for free. At
**zoom 1, no scroll at all**, over 122 real boxes:

| track | | v1.7 (warp) | v1.10 (linear + markers) |
|---|---|---|---|
| 900px | median box width | 100px | **177px** |
| | boxes showing their full label | 31% | **58%** |
| 1400px | median box width | 155px | **274px** |
| | boxes showing their full label | 52% | **72%** |

Box widths roughly double and legibility nearly doubles **before** the zoom
control is touched. Zoom exists to close the remaining gap on demand.

## Implementation Decisions

**The marker threshold is not a sensitive knob.** Replayed at 1%, 1.5%, 2%
and 3%, the box/marker split moves by at most one box across all 28 runs —
real runs are naturally bimodal (a node is either ~0% or >3%; nothing lives
in between). `MARKER_MAX_WIDTH_PCT = 2` is chosen, and the flatness is the
justification, not the value itself.

**Marker eligibility is by width only — no exceptions by kind.** v1.7 (via
fe-19) exempted gate-kind blocks from collapsing, to keep their verdict chips
visible. That exemption is retired for *markers*, for two reasons: the
replay finds 10 genuinely instant gate blocks (`baseline` at 0.00% — a
baseline that ran nothing), which an exemption would inflate for no
information; and fe-19's concern was raised in a world with no panel. v1.7
built the rail and the panel, so a verdict is one click away by construction.
A gate marker still carries its worst-outcome colour, so a failed instant gate
is a red mark on the chart, not an invisible one.

The exemption is **kept for boxes**: a gate drawn as a box still renders its
full verdict strip and still never absorbs a neighbour into a cluster.

**Markers cluster; boxes do not.** A maximal run of consecutive, same-lane
markers with no box between them collapses to one marker. This is
gap-independent, unlike v1.7's gap-aware rule — which only made sense when
small boxes were being widened. It is what structurally prevents fe-19's
stacking: two separate markers in one lane are always separated by a real box.

**Known, accepted consequence.** An agent node that is genuinely brief
renders as a marker. One case exists in 28 runs (`test`, 29.3s inside a 33m
run, 1.46%). It sits alone on its own named lane beside its model, and the
rail carries its duration — it is a dot on a labelled row, not a loss. Zoom
is the remedy when it matters.

## User Stories

1. As an operator, I want a step that took twice as long to be drawn twice as
   wide, so that I can read where the time went without checking a tooltip.
2. As an operator, I want the chart's proportions to agree with the `share`
   meter in the lane rail beside it.
3. As an operator, I want instant steps drawn as marks rather than boxes, so
   that they stop consuming the width the real steps need.
4. As an operator, I want every mark to still be a rail entry I can click, so
   that nothing disappears from the timeline by being small.
5. As an operator, I want the whole run visible on one screen by default, so
   that I can see its shape before reading its detail.
6. As an operator, I want to zoom in and scroll horizontally when I want to
   read a crowded stretch, so that legibility is my choice and not a
   distortion applied on my behalf.
7. As an operator, I want my zoom and scroll position to survive a live tick,
   so that watching a running job does not keep throwing me back to the start.

## Out of Scope

- Vertical (lane) virtualisation — lane counts are small and bounded.
- Any change to `buildGanttView`'s pairing, lane assignment or `GanttView`
  shape. This amendment is entirely downstream of the projection.
- The rail's own ordering and content (v1.7, unchanged).
