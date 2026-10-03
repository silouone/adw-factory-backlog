# Amendment v1.7 — subscription headroom on the operator's screens

> **Status:** proposed 2026-09-17, **awaiting operator approval**.
> **Amends:** `adw-v1.2-live-view.md` — the screens render run-derived facts
> only; this adds the first **account-level** fact. Does not touch
> `constitution.md`. **Depends on:** `adw-fe-14-grid-screen-console-design`
> (the board header this hangs off) — landed.
> **Implemented by:** `tickets/adw-usage-01-…`, `-02-…`, `-03-…`.
> **Prototyped by:** `src/web/usage.prototype.ts` +
> `src/web/USAGE-PROTOTYPE-NOTES.md`, operator review 2026-09-17,
> **variant B picked**.

## Problem Statement

The operator dispatches runs against a Claude subscription with two hard
ceilings: a **5-hour session window** and a **weekly window** (with a separate
per-model weekly figure). Both are invisible from the factory.

Today the only way to see them is to stop, open a terminal, and run
`claude -p "/usage"`. So the question *"can I afford to dispatch this feat
run, or am I about to burn the last of my week on it?"* is answered either by
guessing or by breaking flow — and it is asked before **every** dispatch,
because a median green cLens run is 11.2 minutes and a feat run can be an
hour.

This is not a run-level fact and no journal will ever carry it. It moves while
nothing is running, it counts the operator's own Claude Code sessions, and it
counts machines the factory never saw.

## Solution

Two KPIs are **permanently visible** in the board header — the 5-hour session
and the week (all models) — each as a percent used, a micro-bar, and a live
countdown to its reset. Clicking them pins a panel carrying **everything**
`/usage` reports: the third gauge (per-model week), percent remaining and
reset moment for all three, and the full "what's contributing" breakdown —
requests, sessions, context-size share, long-session share, top skills and top
MCP servers, for both `Last 24h` and `Last 7d`, laid out so the two periods
can be read against each other.

The operator answers "can I afford this run?" without leaving the board, and
"why did I burn so much?" one click deeper.

## User Stories

1. As the operator, I want the 5-hour session percentage visible at all times
   on the board, so that I can decide whether to dispatch now or wait for the
   window to reset.
2. As the operator, I want the weekly (all models) percentage visible at all
   times, so that I know whether I am pacing a week or about to end one.
3. As the operator, I want a live countdown to each window's reset, so that
   "wait for the reset" is a decision with a number attached rather than a
   vague intention.
4. As the operator, I want the percentage rendered as **used**, never as
   remaining, so that it matches what the CLI says and I never misread a
   gauge in the dangerous direction.
5. As the operator, I want the percent remaining printed next to the percent
   used, so that I do not do the subtraction in my head while deciding.
6. As the operator, I want a colour that changes as a window fills, so that
   "nearly out" is legible at a glance without reading the number.
7. As the operator, I want to click a KPI and see the full usage detail, so
   that I do not need a terminal to answer a usage question.
8. As the operator, I want the panel to stay open while I read it, so that a
   live run's SSE push does not close it under my cursor.
9. As the operator, I want the panel to update itself while open, so that a
   figure I am reading is not silently stale.
10. As the operator, I want to see the per-model weekly gauge in the panel, so
    that I know whether a specific model's budget is the binding constraint
    rather than the overall one.
11. As the operator, I want request and session counts for the last 24h and
    the last 7d side by side, so that I can see whether my usage is trending
    up or down.
12. As the operator, I want the two periods aligned on the same labelled rows,
    so that comparing them is a horizontal read rather than a hunt.
13. As the operator, I want the share of usage at large context sizes, so that
    I can connect a burn rate to a working habit I can actually change.
14. As the operator, I want the share of usage from long-running sessions, for
    the same reason.
15. As the operator, I want my top skills and top MCP servers by usage share,
    so that I can see which tools are expensive.
16. As the operator, I want the "approximate, this machine only" caveat
    attached to the contributing section **and nowhere else**, so that I do
    not wrongly distrust the three gauges, which are account-level truth.
17. As the operator, I want to know how old the reading is, so that I can tell
    a live figure from a stale one.
18. As the operator, I want the reading refreshed automatically, so that I do
    not reload the page to trust it.
19. As the operator, I want the same KPIs on the run screen, so that "how much
    have I got left" is answerable while I am watching a run rather than only
    from the board.
20. As the operator, I want the usage read to cost me no tokens, so that
    checking my budget does not consume it.
21. As the operator, I want the read cached, so that opening the board does
    not spawn a process per second per browser tab.
22. As the operator, I want the board to render normally when the `claude` CLI
    is missing or fails, so that a usage widget can never take the board down.
23. As the operator, I want an explicit "unavailable" state rather than a zero
    when the read fails, so that the screen never tells me I have a full week
    left when it simply does not know.
24. As the operator, I want a CLI output-format change to degrade to
    "unavailable" rather than to wrong numbers, so that I find out the parser
    broke instead of trusting it.
25. As the operator, I want a characteristic line the parser does not
    recognise to still be shown, so that a new figure Anthropic adds is not
    silently dropped from my screen.
26. As the operator, I want the project dropdown in the board header to stay
    open when I open it during a live run, so that filtering is usable while
    the factory is working.
27. As the operator, I want a countdown only when it can be computed
    correctly, so that a machine in a different timezone shows me the literal
    reset string rather than a confidently wrong duration.
28. As the operator, I want the whole feature to add no runtime dependency, so
    that `adw-v1.2` decision 11 survives.
29. As a factory developer, I want the usage probe injected, so that the test
    suite never spawns a subprocess and stays deterministic.
30. As a factory developer, I want the prose parser to be pure, so that the
    riskiest part of the feature is the cheapest part to test.

## Implementation Decisions

### D1 — the data source, and why the obvious objection is wrong

The only source on the machine is the `claude` CLI. There is **no** `claude
usage` subcommand and nothing under `~/.claude/` caches the figures (both
checked 2026-09-17).

The probe is `claude -p "/usage" --output-format json`. **Measured**, because
the instinctive objection is *"that burns tokens against the budget it
reports"*:

```
{ "local_command": "usage", "num_turns": 0, "total_cost_usd": 0,
  "duration_api_ms": 0, "usage": { …all zeros… }, "result": "<the prose>" }
wall: 1.82s, 1.85s
```

It is a **local command**: zero tokens, zero turns, zero cost, no API call.
The cost is wall time and a process, not budget.

`--output-format json` rather than plain text is a decision, not a
preference: the plain-text invocation emitted a **settings permission warning
on stderr ahead of the content**. The JSON envelope isolates the prose in
`.result` where no stray stderr line can reach it.

### D2 — ONE seam: an injected probe on `WebServerDeps`

`src/web/` gains the capability to execute a binary. That is a real widening
of a module that is otherwise a pure filesystem reader, and it is admitted
**at exactly one point**, matching the `ServeFn` / `Clock` / `TimerSeam` idiom
already on `WebServerDeps` (Art. IX):

```ts
export interface WebServerDeps {
  // …existing…
  /** Injected, defaulting to the real `claude -p "/usage"` spawn. Returns
   * the CLI's raw stdout. Tests pass a fixture string, so the suite never
   * spawns a subprocess. */
  readonly usageProbe?: () => Promise<string>;
}
```

Rejected alternative, recorded so it is not re-litigated: a separate
`adw usage` command writing a snapshot file, which would keep `src/web/`
subprocess-free. Rejected because freshness then becomes an operator chore
(who runs it, how often) rather than a TTL, and the second moving part buys
nothing the injected seam does not already give the test suite.

### D3 — a pure parser, failing closed

`parseUsage(text, readAt) -> UsageSnapshot | UsageUnavailable` is **pure**:
no I/O, no clock, no randomness. It parses **unversioned CLI prose**, so:

- An unrecognised line is ignored, never fatal.
- **No limit line found at all → `{ok: false, reason}`**, never a snapshot of
  zeros. This asymmetry is the point: a fabricated `0% used` reads as *"you
  have your whole week left"*, which is the single most expensive wrong thing
  this feature could say.

Shape, from the prototype:

```ts
interface UsageGauge {
  key: "session" | "week" | "weekModel";
  label: string; short: string;
  /** percent USED, verbatim from the CLI — never flipped to remaining */
  pct: number;
  resetsAt?: string;   // "Sep 18 at 2:59pm", never reformatted
  resetsTz?: string;   // "Europe/Paris" — load-bearing, see D6
}
interface UsageBehavior {
  raw: string;         // the CLI's own sentence, always
  label?: string;      // present only when a known shape matched
  value?: string;
}
```

### D4 — a 60s TTL cache, and a poll route

`loadUsage` caches process-wide for 60s. The session gauge was observed
moving `11% → 15%` over ~90 minutes, so a minute of staleness is invisible —
while a 1.8s spawn on the 1s SSE tick, per connection, would not be.

A `GET /usage.json` route returns the rendered fragments for the client to
poll on the same cadence. It **404s unless a live variant is requested**, so
the spawn capability is unreachable by default.

### D5 — the three-way split that makes the panel survive SSE

`renderBoardHeader` is called by `renderGridBody`, which `/events` pushes into
`#grid-body.innerHTML`. **Every push destroys the header's DOM.** The run
screen has the identical shape (`renderRunHeader` inside `#run-console`).

So:

| piece | lives | why |
|---|---|---|
| **chips** | inside the header | they belong there visually; stateless, so a swap costs nothing — **re-mounted** by a `MutationObserver` after every push |
| **panel** | outside the swapped fragment | a swap can never close it mid-read |
| **open state** | a closure variable, also outside | re-applied on each re-mount — the idiom `filterScript` already uses |

**Measured on the picked variant:** with the panel open, `hidden` was sampled
every 2s for **218 seconds** across 3+ TTL refreshes and continuous SSE pushes
with two live runs — 29 samples, never once closed, while the figures updated
underneath it.

### D6 — a countdown only when it can be right

`resets Sep 18 at 2:59pm (Europe/Paris)` carries **no year and no UTC
offset**. Computing an epoch from it cross-zone is arithmetic on missing data.

The client renders a countdown **only when the browser's own resolved
timezone equals the printed one**; otherwise the literal string stands alone
and the timezone is printed rather than dropped. The year is the one that
lands the date in `[-1d, +9d]` — the only window a reset can fall in, which
also handles the December→January rollover.

### D7 — presentation: aligned grid, not prose columns

Round 1 rendered each period as an independent prose stack; the two periods
put their rows at different heights and could not be compared, which is the
only reason to show both. The shipped shape is **one grid**: a fixed label
gutter and one column per period, so trend is a horizontal read. A period
missing a metric renders `—`, never a fabricated zero (the `adw-fe-10`
stance).

Aligning requires turning prose into `(label, value)`. The guard: the shape
list is an **alignment aid, never a filter** — an unmatched sentence renders
verbatim in a spill row below the grid, and every gridded cell carries the
CLI's original sentence on hover. *This was validated live by accident: a
characteristic absent from every earlier probe —* `"10% of your usage was
while 4+ sessions ran in parallel"` *— appeared mid-session, matched nothing,
and spilled correctly instead of vanishing.*

### D8 — zero new dependencies

Hand-written CSS and ~60 lines of vanilla JS, no build step. `adw-v1.2`
decision 11 survives intact; this amendment does not reopen D1 of
`adw-v1.5`.

### D9 — the pre-existing defect this uncovered

`es.onmessage` swaps the fragment then calls `apply()`, which restores filter
classes but **not `menu.open`**. The board's project dropdown therefore slams
shut on every SSE tick — i.e. every second with a run in flight. This is a
**defect in shipped code**, independent of this feature in origin, and is
ticketed separately (`adw-usage-01`) because it has a clean red test of its
own and does not need any of this feature to be worth fixing.

It is **not** independent in execution. `adw-usage-01`'s fix and D5's chip
re-mount rewrite the **same inline `filterScript` closure in one file**, so
the tickets are serialised `01 → 02 → 03`. Beyond avoiding a conflict, this is
the right order on merit: 01 establishes the closure-restore idiom that D5
then reuses for the chips.

## Testing Decisions

**A good test here asserts external behaviour**: what the parser returns for a
given CLI output, and what the server responds for a given request. It does
not assert CSS, class names, or the internal shape of a render helper.

| module | seam | prior art |
|---|---|---|
| `parseUsage` | pure function, fixture strings in → snapshot out | `test/intake/ticket.test.ts` (pure parse of an unforgiving external contract) |
| the `/usage.json` route and the `?usage=` render | `startWebServer({ usageProbe })` with a fake probe | `test/web/server.test.ts`, which already injects `serve`, `clock` and `pollTimer` |
| the SSE re-mount / dropdown fix | the rendered fragment and its inline script, asserted as a string | `test/web/render.test.ts`; `adw-fe-16` established the grep-level guard idiom |

Cases that must exist, because each one is a way this feature lies:

- a full, well-formed reading → all three gauges, both periods, every top list
- a reading with **no** limit line → `ok:false`, **never** a zero-filled snapshot
- a characteristic line matching no known shape → present, verbatim, not dropped
- a gauge line with no `resets` clause → renders, with no reset and no countdown
- a reset clause with no timezone → renders, no countdown
- a probe that throws / exits non-zero → `ok:false`, **board still renders**
- `GET /usage.json` without a live variant → 404, **and the probe is not called**
- the TTL: two reads inside the window call the probe **once**

**The suite must never spawn a subprocess.** A test that calls the real
`claude` binary is a flake and a cost; `usageProbe` exists to make that
impossible by construction.

## Out of Scope

- **Any cost or spend figure.** `/usage` reports percentages of a
  subscription allowance, not money. The per-run `$` figures on the cards come
  from `pricing.ts` and are a different quantity; the two must never be mixed
  in one view.
- **Predicting exhaustion.** "At this rate you run out Thursday" needs a
  window start time the CLI does not report. Not inferred, not faked.
- **Gating dispatch on headroom.** This surfaces a number; it never refuses a
  run. Bounded autonomy's ceilings live in the engine, and the operator is the
  scheduler (README, *Deliberately absent*).
- **Per-run attribution of subscription usage.** The journal's `usage`
  records and the account's percentages are different measurements at
  different granularities.
- **Non-subscription auth.** API-key and Bedrock/Vertex users get different
  `/usage` output; out of scope beyond degrading to "unavailable".
- **Multi-machine aggregation.** The contributing section is explicitly
  this-machine-only and is labelled as such.
- **A history of usage over time.** Nothing is persisted; every figure is read
  at render time, mirroring how `metrics.ts` derives rather than stores.

## Further Notes

The prototype is throwaway by construction — `usage.prototype.ts`,
`USAGE-PROTOTYPE-NOTES.md` and the `?usage=` branch are deleted by
`adw-usage-02`, the same lifecycle `render.prototype.ts` (fe-14),
`render-run.prototype.ts` (fe-13), `queue.prototype.ts` and
`summary.prototype.ts` had.

**One thing is recorded rather than diagnosed.** Two manual clicks at
`(1020, 30)` failed to open the fused chip, while a JS probe measured the
button spanning `x 835–1235, y 22–50.7` — the click was *inside* the hit
target. The obvious explanation (a dead gap between the chip's two rows) does
not fit that evidence, and the other candidate (the refresh cycle closing the
panel) was **tested and ruled out** by the 218-second probe in D5. It is an
intermittent input-registration flake with **no established cause**, and
`adw-usage-02` carries it as a reproduce-before-done item. A wrong cause
written down would be worse than an open one.

Variants A and C remain in the prototype for reference but are now laid out
for B's width: C shares B's 660px panel and is unaffected, while A's 470px
panel wraps the skill chips one per line and runs 679px tall.
