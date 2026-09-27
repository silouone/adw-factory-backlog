---
id: adw-fe-20-the-run-panel
type: feat
status: done
priority: 1
created: 2026-09-17
depends: [adw-artifact-01-persist-the-agent-summary]
attempts: [{"runId":"adw-fe-20-the-run-panel-1789607677921","branch":"adw/adw-fe-20-the-run-panel","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-20-the-run-panel-1789607677921/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-20-the-run-panel-1789656576411","branch":"adw/adw-fe-20-the-run-panel-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-20-the-run-panel-1789656576411/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# One reader at the bottom of the run screen, and a timeline that reads as a sequence

> Spec: `specs/adw-v1.7-run-panel.md` — read it first; this ticket is the work
> order, the spec is the contract. Synthesised from six rounds of operator
> review against a live prototype (`?summary=F`,
> `src/web/summary.prototype.ts`), which is deleted when this lands.

## 1. What is wrong

Four defects on one screen, all confirmed against
`adw-perf-03-…-1789588337797`:

1. **The report is not there.** The dock is `hidden` with nothing selected —
   the largest area of the page shows nothing, and the run's own final output
   has no durable source at all (`adw-artifact-01`, this ticket's dependency).
2. **Two panels at once.** With a panel open, clicking a Gantt block opens the
   dock *beside* it. Operator: *"everything is broken in the UI because we got
   2 bottom panel displayed at the same time."*
3. **The timeline does not read as a sequence.** Half the steps are instant
   (`assemble-plan` 0ms, `assemble-build` 1ms, `baseline-green-check` 1ms). On
   a linear axis they are zero-width glyphs with no name or timing; given a
   readable minimum they **overlap their neighbours**. The DOM has 14 `.nd`
   elements where the operator counts 7 boxes — that gap *is* the bug.
4. **Steps that took time claim they took none.** A block's duration is
   `end - start` and always known, but the card reads `metrics.durationMs`,
   which `gantt.ts` documents as undefined for deterministic and gate blocks.
   `dispatch` shows `DURATION —` while the Gantt prints `129ms` in its own
   tooltip.

## 2. The one seam

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

`TimelineLayout` is an ordered list of **drawn boxes** — each covering one or
more `GanttBlock`s, with label, duration, outcome class, covered ids, and
warped `left`/`width` — plus warped tick positions.

**One seam, deliberately.** The rail is one entry per drawn box, so the rail
*is* the layout. Deriving them separately is how they drift apart.

Two constraints that kill the prototype's workarounds:

- The SSE frame is `renderRunHeader + renderLanes(view, now).html +
  renderUpNext`. A **server-side** layout is therefore carried by every frame
  for free — the prototype's `MutationObserver` does not survive into
  production.
- `detail.drawers` is already available server-side, so the panel renders from
  data. The prototype's lifting of markup out of hidden `.card` divs does not
  survive either.

## 3. What to build

Per the spec's *Implementation Decisions* — summarised, not restated:

- **`layoutTimeline`**: cluster instant boxes closer than the minimum readable
  width, order by execution, warp (small boxes get a fixed slice; the rest
  share the remainder in true proportion; laid end to end, normalised to
  100%), and expose `toWarped` for ticks and the gridline overlay.
- **The panel owns the dock**: two scopes (run / step) through one shell. Run
  scope is the initial ticket beside the agent final output, each scrolling
  independently. Step scope is that step's own header and sections.
- **Rail**: overview + one entry per drawn box, execution order, name +
  duration, outcome-coloured.
- **Controls**: Markdown/raw toggle, one close, full-screen collapsing the
  Gantt to a clickable header, `Esc` unwinding step → run → closed.
- **Layout**: single-viewport app shell, panel owns the only scroll region.
  Every grid in the height chain declares explicit rows.
- **Fix the three defects**: duration from `end - start`; every drawn box
  labelled; warped widths summing to 100% make overhang structurally
  impossible.
- **Metric tri-state**: known / pending / not-applicable, so a deterministic
  node's missing token counts stop looking like a pending number.

## 4. Build protocol

TDD, Article I — no source before a reviewed, red test.

1. Write the `layoutTimeline` tests first, from fixture journals — order,
   clustering, proportion, no-overlap, total exactly 100, tick monotonicity
   and containment, the no-nesting invariant guard, duration from
   `end - start`, and the tri-state. Prior art: `test/web/gantt.test.ts` and
   `test/web/chain.test.ts` — both drive a pure projection from a fixture
   journal with no DOM and no clock.
2. Confirm red. Present for operator review.
3. Implement to green.
4. Thin rendering coverage only in `test/web/render-run.test.ts`: one rail
   entry per drawn box, one close control. Pixels are not a test.
5. Delete `src/web/summary.prototype.ts`, `SUMMARY-PROTOTYPE-NOTES.md` and the
   `?summary=` branch in `server.ts`.
6. `bun run lint && bunx tsc --noEmit && bun test` all green.

## 5. Amendment watch

The warp is sound **only because the engine guarantees nodes never nest**
(`runNode`, engine.ts). The implementation must fail loudly if a journal
violates that, never mis-draw. If the invariant turns out to be softer than
the spec assumes: **stop and propose an amendment** — do not silently widen
the layout to cope.

## 6. Out of scope

- `adw-artifact-01` — the dependency, its own ticket.
- `adw-fe-18` (a live *agent* node on the `code` lane) — related, distinct,
  already `blocked`.
- `adw-fe-19` — **superseded**. Its Defect A (instant nodes stacking) is
  subsumed by the warp; its Defect B (blank drawer metrics) by the tri-state.
  Close it as superseded rather than building it.
- Any change to `buildGanttView`'s contract — `capture.ts`, `metrics.ts` and
  the existing web tests depend on it.
- Markdown fidelity: small, dependency-free, offline. No CommonMark
  conformance, no CDN.
