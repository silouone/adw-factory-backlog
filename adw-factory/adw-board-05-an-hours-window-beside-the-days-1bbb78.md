---
id: adw-board-05-an-hours-window-beside-the-days-1bbb78
type: feat
status: queued
priority: 2
created: 2026-09-28
caps: {minutes: 120, turns: 500, stallMinutes: 20}
depends: []
attempts: []
---
# The narrowest board window is a whole day — too wide for one working session

> **Operator, 2026-09-28.** `1D` is the tightest window the board offers,
> and today that is 42 of 168 runs. What is actually wanted is *"this
> session"* — the runs since the wave was fired a couple of hours ago.
>
> **The ask:** `2H` / `5H` / `10H` chips alongside `1D` / `3D` / `7D` /
> `ALL`.

## Correcting the premise before building on it

The operator's note said `1D` "goes back to the day before (human day)".
It does not: `sinceForDays` is `connectMs - Number(daysValue) * DAY_MS`
(`range.ts`), a **rolling 24h from page-connect** — never a calendar
boundary. The ask stands on its own merit (24h is simply wider than a
session); build it as a narrower rolling window, not as a calendar-boundary
bug fix. Say so in the PR body so the change is not later read as one.

## Three things break if the encoding is only widened by hand

`DaysWindow = "1" | "3" | "7" | "all"` is **numeric-by-convention** and
three call sites parse it with `Number()` — `Number("2h")` is `NaN`, and
`NaN` propagates silently into an empty board and an unlit histogram:

1. `sinceForDays` — `connectMs - Number(daysValue) * DAY_MS`;
2. `RangeChips`'s lit-bar rule — `i < Number(activeDays)`;
3. `parseDaysParam` / `withDaysParam` — the `"3"|"7"|"all"` allow-list and
   the `"1"`-means-default rule.

The clean move is to re-base the unit on **hours** and make every window
carry its own unit in the token.

**The server needs nothing.** `?since=` is a raw epoch (`parseSinceParam`,
`server.ts`); a 2-hour window is just a larger number.

## The histogram: decided here, not by the builder

`RangeChips`'s histogram buckets per **calendar-ish day** (`dayIndexOf`,
`DAY_MS`). On a 2-hour window it would collapse to a single bar.

**Decision: the histogram stays daily and unchanged.** It is the *archive
shape* — "how much has this factory run, day over day" — not a readout of
the active window. Only its **lit** rule has to learn about sub-day
windows: any window under 1 day lights **day 0 only**. Do not make the
bucket size follow the window; that is a different feature and it is not
being asked for.

## Requirements

- [ ] **R1 — red first: the window type, `range.ts` /
      `test/web/ui/range.test.ts`.** Widen to
      `Window = "2h" | "5h" | "10h" | "1d" | "3d" | "7d" | "all"`, keeping
      `DaysWindow` as a deprecated alias only if that materially shrinks
      the diff — otherwise rename cleanly and update every reference.
      Add one pure exported helper, `windowMs(w): number | undefined`
      (`undefined` for `"all"`), and express `sinceForDays` in terms of it.
      **No `Number()` call may survive anywhere in this module** — pin that
      with a test asserting each of the seven tokens produces the exact
      expected `since` offset from a fixed `connectMs`.
- [ ] **R2 — red first: URL compatibility, same test file.**
      `parseDaysParam` accepts the seven tokens **plus** the four legacy
      values already live in bookmarked URLs (`"1"`→`1d`, `"3"`→`3d`,
      `"7"`→`7d`, `"all"`), and folds anything else to the default. The
      default stays `1d`. `withDaysParam` omits the key **only** for the
      default and emits the new token otherwise; `eventsUrl` is unchanged
      in shape and passes every other param through untouched. Test the
      legacy values explicitly — this is the one backward-compatibility
      claim in the ticket.
- [ ] **R3 — red first: seven chips, `RangeChips.tsx` /
      `test/web/ui/RangeChips.test.tsx`.** `WINDOWS` becomes the seven
      tokens, rendered in ascending order: `2H 5H 10H 1D 3D 7D ALL`. Chip
      labels uppercase the token (`2h` → `2H`), matching how `1d`/`all`
      already read in the header. They ride the existing `.gh-r .chip`
      rules — **no new chip styling**. Seven chips plus the histogram plus
      the label plus the project dropdown plus four health chips is a wide
      row: confirm the existing `@media(max-width:900px)` `.gh-r` rule
      still wraps sanely, and tighten only `.range`'s gap if it does not.
- [ ] **R4 — red first: the lit rule and the label, same test file.**
      Replace `i < Number(activeDays)` with a rule expressed in
      `windowMs`: a bar at day-index `i` is lit when the window covers any
      part of that day — so `2h`/`5h`/`10h` light **day 0 only**, `1d`
      lights day 0, `3d` days 0-2, `all` lights every bar (unchanged for
      the four existing windows — pin that as a no-regression test). The
      `since <date> · N of M` label keeps its exact `>=` boundary against
      `index.epochs`, which already agrees with the server's own filter;
      for a sub-day window the date alone is ambiguous, so render the time
      too (`since Sep 28 14:20 · 6 of 168`), using the same deterministic
      UTC-based formatter `monthDay` already establishes — **not**
      `Intl.DateTimeFormat`, for the reason that function's own doc-comment
      gives.
- [ ] **R5 — the call sites, no behaviour change.** `entry.tsx`
      (`parseDaysParam` on load) and `router.ts` (`eventsUrl` /
      `withDaysParam`, the reopen-on-change effect) compile against the new
      type with no logic change — changing the window still closes the
      stream and reopens it against the recomputed `since`, exactly as
      today.
- [ ] **R6 — out of scope, deliberately.** No "since the first run of my
      session" adaptive window (a real and different idea — it belongs in
      its own ticket). No per-window histogram bucketing (see the decision
      above). No run-screen change — the run screen has no window concept.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`:
`2H` shows only the current session's runs with a time-stamped `since`
label, `1D` matches today's board exactly, and an old `?days=3` URL still
loads the 3-day window.
