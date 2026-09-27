---
id: adw-bug-02-run-screen-axis-and-liveness
type: bug
status: blocked
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-bug-02-run-screen-axis-and-liveness-1789407239489","branch":"adw/adw-bug-02-run-screen-axis-and-liveness","workspace":"/Users/silouane/adw-factory/runs/adw-bug-02-run-screen-axis-and-liveness-1789407239489/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-bug-02-run-screen-axis-and-liveness-1789738849699","branch":"adw/adw-bug-02-run-screen-axis-and-liveness","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-02-run-screen-axis-and-liveness-1789738849699/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# A run that never ended renders a 61-day time axis, and the run screen never refreshes while it watches one

> **Rescoped 2026-09-18 by ledger audit — narrowed to Defect A.**
> Re-verified against current `src/web/`, not against memory:
>
> | defect | state |
> |---|---|
> | **A** — `block.end ?? now` stretches a dead run's axis to wall-clock | **STILL LIVE** — `src/web/timeline.ts:168`, no liveness guard. `adw-fe-20` moved `timeBoundsOf` here and `adw-fe-22` retired the warp; neither touched the dead-run bound. |
> | **B** — tick labels use the wrong formatter | landed — the `"1m 60s"` zero-pad fix is in `src/web/timeline.ts:143` |
> | **C** — the run screen never updates while you watch | landed by `adw-fe-16` — `EventSource("/run-events")`, `src/web/render-run.ts:522` |
>
> Build **only Defect A and its §4 axis-window policy**. Sections 2 and 3 below
> are kept as the record of what was observed, and are already discharged.

> Spotted by the operator 2026-09-14 against two live screens. Three defects in
> `src/web/render-run.ts`, all independent of `adw-bug-01`, all reproducible
> from banked journals.

## 1. Defect A — `block.end ?? now` stretches the axis to wall-clock

`timeBoundsOf` sizes an unterminated block against `now`:

```ts
const end = block.end ?? now;   // render-run.ts
```

For a **live** run that is correct — the open block really is still running.
For a run that died without a `run-end`, the last `node-start` never got its
`node-end`, so `max` becomes *the current time*, and the axis spans from the
run's start to whenever you happen to open the page.

Measured across all 115 banked runs, 2026-09-14:

| | |
|---|---|
| runs whose axis is stretched **>2×** | **11 of 115** |
| worst | `m4s-006` — 0.0m of real work rendered on an **83,739-minute** axis (**3,103,358×**) |
| the operator's screenshot | `adw-fe-03-prompt-persistence` — ~3m of work, axis to `1200m 0s` |

Every block collapses to a sliver at the far left, labels overlap into an
unreadable smear, and the `share` meter reports **100%** for the lane holding
the open block — because that block "ran" for 19 hours.

**The fix must not break live runs.** `projectRun` already answers this with
the three-state liveness model (`running` / `finished` / `unknown`, from
heartbeat freshness, `adw-fe-08`):

- [ ] `running` → keep sizing the open block against `now`. That is the point.
- [ ] `finished` / `unknown` → clamp to the **last timestamp the journal
      actually contains**, never `now`.
- [ ] Same clamp in `laneShare` — it has the identical `(b.end ?? now)` and the
      identical bug.
- [ ] An open block on a non-live run must still be **visibly** open — it did
      not finish, and the screen must not imply it did. Clamping its extent is
      not the same as claiming an end time.

## 2. Defect B — the tick labels are `fmtMs`, which is the wrong formatter

`ticksFor` labels every tick with `fmtMs`, a *duration* formatter. On a
round-numbered axis that always produces the noise word `0s`:

```
span    0.3m  ->  0ms   5.0s   10.0s  15.0s
span    3.0m  ->  0ms   30.0s  1m 0s  1m 30s  2m 0s
span   60.0m  ->  0ms   10m 0s 20m 0s 30m 0s 40m 0s
span 88285m   ->  0ms   60m 0s 120m 0s 180m 0s 240m 0s
```

- [ ] `0ms` at the origin — it is an axis origin, not a duration. `0s`.
- [ ] `1200m 0s` instead of `20h`. Minutes never roll into hours, so a long
      axis prints four-digit minutes.
- [ ] `10.0s` — a tenth of a second is meaningless on an axis whose step is 5s.
- [ ] Give the axis its **own** formatter: `0s` · `30s` · `5m` · `1h 30m`.
      `fmtMs` stays as-is for block durations, where `4.5s` is right.

## 3. Defect C — the run screen never updates while you watch a run

`render-run.ts` contains **zero** `EventSource`. `/events` pushes only
`renderGridBody` — the grid. So the one screen an operator opens *to watch a
run in progress* is static HTML, and the operator's second screenshot shows
exactly that: `adw-bug-01` live, `DUR —`, `TURNS 0`, frozen at `provision`.

`adw-fe-11-live-bars` (`status: blocked`, `type: manual`) is where live bars
were scoped and it was **never built** — so this is a known gap, not a
regression. This ticket does not build it.

- [ ] **In scope:** make the page not lie while it is stale. Either a `<meta
      http-equiv="refresh">`-class reload for a run whose state is `running`,
      or a small `/events`-style poll for the single run — whichever is
      honestly the smaller change.
- [ ] **A live run must never render as a dead one.** If liveness cannot be
      shown, the page must say it is a snapshot and when it was taken.
- [ ] **Out of scope:** per-block live growth, the streaming bars of
      `adw-fe-11`. Say so in the PR body; do not quietly absorb that ticket.

## 4. The axis window policy the operator asked for

Today the axis fits the data exactly, so a run 15 seconds old gets a 15-second
axis that is meaningless a minute later, and a live run's axis **rescales on
every render** — every block moves even though nothing about them changed.

- [ ] **A minimum window at load.** A run that has been going 15s renders
      against a floor (suggest **5m**), not against 15s.
- [ ] **Growth in quantised steps, not continuously.** When the run outgrows
      the window, jump to the next step — `5m → 10m → 15m → 30m → 1h → 2h → …`
      — so the axis is stable between steps and a block's position means
      something from one look to the next.
- [ ] **Never shrink below the data.** The window is `max(floor, next step ≥
      real span)`.
- [ ] The step table is named and commented, like `ticksFor`'s `nice` array
      already is.

## 5. Red tests — Article I

- [ ] **Defect A:** a hand-built `GanttView` with one open block, a `now` far
      past the last journal timestamp, and a **non-live** state → the computed
      span must be the journal's span, not `now - start`. Assert the number.
- [ ] **Defect A, the other half:** the same view with a **live** state → the
      span *does* extend to `now`. Both directions, or the fix is a regression
      waiting to happen.
- [ ] **Defect B:** assert the exact label strings for a handful of spans.
      `0s`, not `0ms`; `20h`, not `1200m 0s`.
- [ ] **§4:** a 15-second run yields the 5m floor; a 6-minute run yields the
      10m step; a run never yields a window smaller than its own span.
- [ ] Red before any fix. `red-check` classifies them.

## Verify

- [ ] `m4s-006-1784377213484` (the 3,103,358× case) renders a legible axis.
- [ ] `adw-fe-03-prompt-persistence`'s run renders ~3m of work across the full
      width, and its `share` meter no longer reads 100% for the open lane.
- [ ] A genuinely live run still grows toward `now`.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Note for `adw-fe-14`

The grid prototype's card chart computes its own bounds the same way
(`block.end ?? hi`) and inherits Defect A. Whichever lands second should use
the shared, fixed bounds function rather than a second copy (Art. VIII).
