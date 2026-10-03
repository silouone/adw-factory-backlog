# Amendment v1.7 — the run panel: one reader, and a timeline that reads as a sequence

> **Status:** proposed 2026-09-17, synthesised from six rounds of operator
> review against a live prototype (`?summary=F`, kept in
> `src/web/summary.prototype.ts` + `SUMMARY-PROTOTYPE-NOTES.md` until this
> lands).
> **Amends:** `adw-v1.2-live-view.md` §D (the projection) and its dock
> contract. **Binding:** `constitution.md` — unchanged.
> **Depends on:** `adw-artifact-01-persist-the-agent-summary` (the overview's
> right-hand pane has no durable source without it).
> **Implemented by:** `tickets/adw-fe-20-the-run-panel.md`.

## Problem Statement

The operator opens the run screen to answer two questions: *what did this run
do*, and *why did it take that long*. Today the screen answers neither well.

**The final report is not there at all.** When a run ends, the agent's summary
is printed to the terminal and embedded in the PR body — then dropped. The
screen whose whole job is to show a run cannot show the one artefact the run
produced. The operator's own words: *"we got an output in the console, which
we probably shouldn't… but we need it also in the web app."*

**The bottom of the screen is dead space until you click, and broken after.**
The dock is `hidden` with nothing selected — the most valuable real estate on
the page shows nothing. And with a panel open, clicking a Gantt block opened a
*second* panel beside the first: *"everything is broken in the UI because we
got 2 bottom panel displayed at the same time."*

**The timeline lies about sequence.** A run is a strictly ordered chain, but
half its steps are instant (`assemble-plan` 0ms, `assemble-build` 1ms,
`baseline-green-check` 1ms). On a linear axis they have zero width and render
as bare glyphs — no name, no timing. Given a readable minimum width they
*overlap their neighbours*, so `assemble-build` sits on top of where `plan`
ends and `build` begins. Either way the lane stops reading as the sequence it
is.

**Steps that plainly took time claim they took none.** `dispatch` shows
`DURATION —` in its card while the Gantt prints `129ms` in its own tooltip two
inches above.

## Solution

One panel at the bottom of the run screen, always open, that the Gantt
**refocuses** rather than duplicates.

With nothing selected it shows the run: the **initial ticket** beside the
**agent final output**, two panes side by side, each scrolling on its own, so
what was asked can be read against what came back. Selecting any step swaps
the same panel to that step's own header and sections.

A left rail lists the run overview plus **one entry per box drawn on the
Gantt**, in execution order, each with its name and duration — so the rail is
a table of contents for the pipeline and can never disagree with the chart
above it.

The time axis becomes **piecewise-linear**: long nodes keep their true
proportions, instant nodes get a fixed readable slice, and boxes are laid end
to end. The lane reads as the chain it is. Ticks and gridlines are remapped
through the same function, so a label still sits above the moment it names.

The panel reads Markdown or raw, collapses the Gantt to its header for
full-screen reading, and closes from one control.

## User Stories

1. As an operator, I want the agent's final report rendered in the web app, so
   that I do not have to scroll back through a terminal that has moved on.
2. As an operator, I want the initial ticket beside that report, so that I can
   judge whether the agent did what was asked.
3. As an operator, I want both panes to scroll independently, so that I can
   compare a point in the ticket against a point in the report without losing
   either.
4. As an operator, I want the bottom panel open by default, so that the
   largest area of the screen is never blank.
5. As an operator, I want clicking a Gantt block to refocus the existing
   panel, so that I never see two panels stacked.
6. As an operator, I want a breadcrumb showing `run ▸ <step>`, so that I know
   which scope I am reading and can get back in one click.
7. As an operator, I want `Esc` to step out of a node to the run, and out of
   the run to a closed panel, so that one key unwinds the whole state.
8. As an operator, I want a left rail listing every step of the pipeline, so
   that I can reach any step without hunting for its box on the chart.
9. As an operator, I want the rail in execution order, so that it reads as the
   run's story rather than as an artefact of lane grouping.
10. As an operator, I want each rail entry to carry its duration, so that I can
    see where the time went before clicking anything.
11. As an operator, I want exactly one rail entry per box I can see on the
    Gantt, so that the two views never disagree about how many steps there
    were.
12. As an operator, I want a rail entry to be colour-coded by outcome, so that
    a failed step is findable without reading every row.
13. As an operator, I want one section per step rather than a stack of
    sub-panels, so that selecting a step shows its content, not a menu.
14. As an operator, I want every instant step to appear on the Gantt with its
    name and duration, so that a 1ms step is a step and not an unlabelled
    tick.
15. As an operator, I want no two boxes to overlap, so that the lane reads as
    a sequence.
16. As an operator, I want long nodes to keep their relative sizes, so that
    the chart still tells me `plan` took longer than `build`.
17. As an operator, I want the axis labelled as non-linear when it is warped,
    so that I am not misled into reading widths as durations.
18. As an operator, I want tick labels and gridlines to stay aligned with the
    boxes, so that the axis remains truthful after warping.
19. As an operator, I want no box to be drawn outside the track, so that
    nothing is clipped at either edge.
20. As an operator, I want a step's real duration shown in its detail, so that
    a step that took 129ms does not claim it took none.
21. As an operator, I want metrics that can never apply to a deterministic
    step to look different from metrics that are merely pending, so that I do
    not wait for a number that is never coming.
22. As an operator, I want to toggle between rendered Markdown and raw text,
    so that I can read a report comfortably but still inspect it verbatim.
23. As an operator, I want one close control on the panel, not one per
    section, so that closing is unambiguous.
24. As an operator, I want a full-screen mode that collapses the Gantt to its
    header, so that a long report gets the whole window.
25. As an operator, I want the collapsed header to be clickable, so that the
    way back is visible rather than guessed.
26. As an operator, I want full-screen to survive navigating between steps, so
    that reading several reports does not mean re-expanding each time.
27. As an operator, I want the page to fit the window with no page-level
    scrollbar, so that I am not scrolling in three directions to read one
    thing.
28. As an operator, I want the panel to absorb the space the Gantt does not
    use, so that a short run gives me a tall reader.
29. As an operator, I want the reader to fill its pane even when the content
    is short, so that the panel does not end in a band of dead space.
30. As an operator, I want the panel to survive live SSE refreshes, so that a
    running run does not reset what I am reading.
31. As an operator watching a live run, I want the rail and timeline to update
    as steps complete, so that the panel is useful before the run ends.
32. As an operator, I want the run KPIs shown once, so that the same four
    numbers are not repeated a few hundred pixels apart.
33. As an operator whose workspace has been reclaimed, I want the panel to say
    the report is unavailable and why, so that a blank pane is never confused
    for an empty report.
34. As an operator of a `blocked` run, I want its agent output too, so that I
    can read the blocker without opening a PR that was never created.
35. As a future maintainer, I want the rail and the Gantt derived from one
    projection, so that they cannot drift apart.
36. As a future maintainer, I want the warp computed server-side, so that the
    client does not re-derive layout on every SSE frame.

## Implementation Decisions

### One new seam: `layoutTimeline`

A pure function over the existing `GanttView`, producing the geometry **and**
the rail:

```
buildGanttView(journal)  ->  GanttView            (exists, unchanged)
        |
        v
layoutTimeline(view, now) -> TimelineLayout        (NEW, pure)
        |                           |
        v                           v
   renderLanes                 renderRunPanel
   (boxes + warped ticks)      (rail = the same boxes)
```

`TimelineLayout` carries an ordered list of **drawn boxes** — each one or more
`GanttBlock`s, with a label, a duration, an outcome class, the ids it covers,
and its warped `left`/`width` — plus the warped tick positions.

This is deliberately **one** seam. The rail is one entry per drawn box, so the
rail *is* the layout; deriving it separately is how the two drift apart.

### The warp

From the prototype, the decision-carrying part:

```
boxes sorted by start
  small := realWidth < minReadableWidth
  fixed := sum(minReadableWidth for small boxes)
  room  := 100 - fixed
  each small box: width = minReadableWidth
  each other box: width = (realWidth / sumRealWidthOfOthers) * room
  laid end to end; normalise so the total is exactly 100
```

and a `toWarped(realFraction)` that maps any real position into warped space
by locating its segment and interpolating — applied to tick labels and the
gridline overlay so they stay aligned.

The warp is sound **only because the engine guarantees nodes never nest**
(`runNode`, engine.ts). If that invariant is ever relaxed, this collapses; the
implementation must fail loudly rather than mis-draw.

### Clustering

Instant boxes closer together than the minimum readable width collapse into
one drawn box, labelled `<first> +N`, carrying every member's id. Clicking it,
or any of its members, selects the same rail entry. This is what makes "one
rail entry per box you can see" true.

### The panel owns the dock

`renderRunPage`'s dock stops being an empty container filled by a click
handler, and becomes the panel: rail plus reader, rendered server-side from
`TimelineLayout` and the already-available drawer views. Two scopes — run and
step — through one shell.

The prototype achieved this with a capture-phase listener that pre-empted the
page's own handler, and a `MutationObserver` to survive SSE swaps. **Neither
survives into production.** The SSE frame is `renderRunHeader + renderLanes +
renderUpNext`, so a server-side layout is carried by every frame for free.
Likewise the prototype lifted card markup out of hidden `.card` divs; the
server already has `detail.drawers`, so the panel renders from data.

### Three defects fixed in passing

1. **Duration.** A block's duration is `end - start`, always known. The card
   currently reads `metrics.durationMs`, which `gantt.ts` documents as
   undefined for deterministic and gate blocks — so instant steps show `—` for
   a number the Gantt already prints in its own tooltip.
2. **Unlabelled markers.** A zero-width block renders as a bare glyph. Every
   drawn box carries its name and duration.
3. **Overhang.** A block near the end of a run can be positioned such that it
   extends past the track. With warped widths summing to 100% this becomes
   structurally impossible rather than clamped.

### Metric tri-state

`—` currently means three different things. Split them: **known**, **pending**
(an agent node mid-flight), and **not applicable** (a deterministic or gate
node, which will never have turns or tokens). Only the third is a new claim;
it is already knowable from `kind`.

### Layout contract

The run screen becomes a single-viewport app shell: the Gantt at its natural
height, the panel taking the remainder and owning the only scroll region. Any
grid in that chain declares explicit rows — a `height:100%` child of an
auto-sized row resolves to its own content height, which is what produced the
dead band the operator photographed twice.

### KPIs

`renderRunHeader` keeps DUR/TURNS/BILLED/CONTEXT. The panel does not repeat
them; it shows only what the header cannot — which scope is being read.

## Testing Decisions

A good test here asserts **external behaviour of the projection**, never
markup. `layoutTimeline` is a total function of `(GanttView, now)`, so every
interesting case is a fixture journal and a returned structure — no DOM, no
clock, no snapshots of HTML.

Tested at the seam, mirroring `test/web/gantt.test.ts` and
`test/web/chain.test.ts` (the closest prior art — both drive a pure projection
from a fixture journal):

- **Order** — a journal whose lanes are interleaved yields boxes in execution
  order, not lane order.
- **Clustering** — consecutive instant nodes within the minimum width collapse
  to one box carrying all their ids; nodes further apart do not.
- **Proportion** — given two long nodes, their warped widths stay in the same
  ratio as their real durations.
- **No overlap** — for any fixture, no two boxes' warped ranges intersect, and
  the total is exactly 100.
- **Tick alignment** — `toWarped` maps a tick inside a box to a position
  inside that box's warped range; monotonicity holds across the whole run.
- **Invariant guard** — a journal with overlapping blocks (violating the
  engine's no-nesting rule) fails loudly.
- **Duration** — an instant deterministic block reports its real
  `end - start`, not `undefined`.
- **Tri-state** — a deterministic block reports token metrics as *not
  applicable*; an in-progress agent block reports them as *pending*.

`test/web/render-run.test.ts` covers the rendering thinly — that the panel
emits one rail entry per drawn box and one close control — and nothing more;
pixel behaviour is not a test.

## Out of Scope

- **Persisting the agent summary.** Its own ticket
  (`adw-artifact-01-persist-the-agent-summary`), declared as a dependency.
  Until it lands the right-hand pane renders its "not available" state.
- **`adw-fe-18`** — a live *agent* node drawn on the `code` lane and labelled
  deterministic. Related but distinct; its own ticket, already `blocked`.
- **`adw-fe-19`** — this amendment subsumes its Defect A (instant nodes
  stacking) via the warp, and its Defect B (blank drawer metrics) via the
  tri-state. That ticket should be closed as superseded rather than built.
- The D and E prototype variants, and the `?summary=` branch — deleted when
  this lands.
- Any change to `buildGanttView`'s contract. `capture.ts`, `metrics.ts` and
  the existing web tests depend on it.
- Markdown rendering fidelity. A small dependency-free renderer is enough; no
  CommonMark conformance, no external library (the page must work offline).

## Further Notes

The prototype lives at `src/web/summary.prototype.ts` behind `?summary=F`,
with `SUMMARY-PROTOTYPE-NOTES.md` recording the six review rounds and the
variants that lost (A stacked, B three-column, C tabs, D focus-split columns,
E focus-stack rows). Both are deleted when this lands — they are the working
drawing, not the building.

Two things the prototype proved that prose would not have: the operator's
count of "7 boxes" against the DOM's 14 `.nd` elements is what surfaced the
clustering requirement, and the overlap only became visible once instant nodes
were given a readable minimum width. Neither was predictable from the code.

The warp is a real trade — the axis stops being uniformly scaled. It is taken
deliberately, on the grounds that a chain that reads as a chain is worth more
than an axis nobody measures with, and it is labelled in the UI so the trade
is visible to whoever reads it next.
