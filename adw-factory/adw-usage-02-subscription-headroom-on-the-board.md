---
id: adw-usage-02-subscription-headroom-on-the-board
type: feat
status: done
priority: 1
created: 2026-09-17
caps: {minutes: 150, turns: 700}
depends: [adw-usage-01-the-header-dropdown-closes-itself-during-a-live-run]
attempts: [{"runId":"adw-usage-02-subscription-headroom-on-the-board-1789607738320","branch":"adw/adw-usage-02-subscription-headroom-on-the-board","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-02-subscription-headroom-on-the-board-1789607738320/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/65","provider":"claude","model":"sonnet"}]
---
# The board never says how much subscription is left

> Spec: `specs/adw-v1.7-subscription-headroom.md`. Design settled by
> `src/web/usage.prototype.ts` — **variant B**, operator review 2026-09-17.
> Read `src/web/USAGE-PROTOTYPE-NOTES.md` before starting: it carries the
> measurements this ticket depends on, and the design is already decided.

## 1. What the operator cannot do today

Answer *"can I afford to dispatch this feat run?"* without leaving the board.
The 5-hour session and weekly ceilings are invisible from every factory
screen; the only way to see them is a terminal and `claude -p "/usage"`. The
question is asked before every dispatch.

## 2. What lands

Two KPIs **permanently visible** in the board header — **5-hour session** and
**week (all models)** — each a percent used, a micro-bar and a live countdown
to reset. Clicking pins a panel with everything `/usage` reports. Variant B:
one fused meter chip, click to pin, `Esc` or ✕ to close.

The complete payload is spec §"User Stories" 1–18 and §D7. Do not re-derive
the layout — the prototype renders it and the operator approved it.

## 3. The four things that are decided, and must not drift

**D1 — the probe is free.** `claude -p "/usage" --output-format json` is a
**local command**: `num_turns: 0`, `total_cost_usd: 0`, `duration_api_ms: 0`,
measured 1.82s/1.85s wall. Zero tokens. Use `--output-format json`, not plain
text: the plain-text path emits a settings permission warning on **stderr
ahead of the content**, and the envelope isolates the prose in `.result`.

**D2 — ONE seam.** `usageProbe?: () => Promise<string>` on `WebServerDeps`,
beside the existing `serve` / `clock` / `pollTimer`, defaulting to the real
spawn. This is the only point at which `src/web/` gains the capability to
execute a binary. **The suite must never spawn a subprocess.**

**D3 — the parser is pure and fails closed.**
`parseUsage(text, readAt) -> UsageSnapshot | UsageUnavailable`. No limit line
found → `{ok:false, reason}`. **Never a zero-filled snapshot** — a fabricated
`0% used` reads as "you have your whole week left", the most expensive wrong
thing this screen could say. `pct` is USED, never flipped to remaining.

**D5 — the three-way split is the load-bearing part.** `adw-usage-01` lands
first and is a **hard dependency, not a courtesy**: both tickets rewrite the
same inline `filterScript` closure in `src/web/render.ts`, so run in parallel
they would conflict and whichever PR landed second would have been gated
against a base missing the other's change. 01 also establishes the
closure-restore idiom this decision reuses — inherit it, do not reinvent it.

The header is inside
the SSE fragment, so every push destroys it. Chips go **inside** and are
re-mounted after each swap; the panel lives **outside** the swapped fragment;
open state lives in a closure **outside** it and is re-applied on re-mount.
Verified in the prototype: panel open across 218s / 29 samples / 3+ refreshes
with two live runs, never closed.

Also non-negotiable, each for a stated reason in the spec:

- the "approximate, this machine only" caveat is scoped to the **contributing
  section only** — it does not qualify the three gauges (§D7, story 16)
- a countdown renders **only** when the browser's zone matches the printed
  one; otherwise the literal string and its zone stand (§D6)
- an unrecognised characteristic line renders **verbatim**, never dropped
  (§D7 — this was validated live, by accident)
- 60s TTL cache; `GET /usage.json` **404s** without a live variant so the
  spawn is unreachable by default (§D4)
- zero new runtime dependencies (§D8)

## 4. Red tests first (Art. I)

Prior art: `test/intake/ticket.test.ts` for the pure parse of an unforgiving
external contract; `test/web/server.test.ts`, which already injects `serve`,
`clock` and `pollTimer`, for the route.

Every case below is a distinct way this feature can lie:

1. a full well-formed reading → three gauges, both periods, every top list
2. **no limit line at all → `ok:false`**, not a zero-filled snapshot
3. a characteristic matching no known shape → present, verbatim, not dropped
4. a gauge line with no `resets` clause → renders, no reset, no countdown
5. a reset clause with no timezone → renders, no countdown
6. the probe throws → `ok:false` **and the board still renders**
7. the probe exits non-zero → `ok:false` **and the board still renders**
8. `GET /usage.json` with no live variant → 404 **and the probe is not called**
9. two reads inside the TTL → the probe is called **once**
10. absent `?usage=`-equivalent, the board's HTML is unchanged from today

Assert external behaviour only — what the parser returns and what the server
responds. Not CSS, not class names, not the shape of a render helper.

## 5. Delete the prototype

This ticket **removes** `src/web/usage.prototype.ts`,
`src/web/USAGE-PROTOTYPE-NOTES.md`, the `?usage=` branch, the `/usage.json`
prototype route and the widened `handleRequest` return type comment — folding
the winner into `render.ts` / `server.ts` properly. Same lifecycle
`render.prototype.ts` (fe-14), `render-run.prototype.ts` (fe-13),
`queue.prototype.ts` and `summary.prototype.ts` had.

`handleRequest` must still return `Response | Promise<Response>` for real
(the probe is async) — that part stays; only the "delete me" comment goes.

## 6. One known unknown, to reproduce before calling this done

Two manual clicks at `(1020, 30)` failed to open the prototype's fused chip,
while a JS probe measured the button spanning `x 835–1235, y 22–50.7` — the
click was **inside** the hit target. The obvious explanation (a dead gap
between the chip's two rows) **does not fit that evidence**, and the refresh
cycle was ruled out by the 218s probe. Cause **not established**.

Reproduce it against the real implementation before done. If it persists,
make the chip's inner rows non-interactive children of one hit target and say
whether that fixed it. **Do not write down a cause you have not verified.**

## 7. Out of scope

Spec §"Out of Scope" in full — in particular: no cost/spend figure (this is
percent of allowance, not money, and must never mix with the cards' `$`), no
exhaustion prediction, no gating of dispatch, no persistence/history, and the
run screen, which is `adw-usage-03`.

## 8. Verify

- `bun run lint && bunx tsc --noEmit && bun test` all green
- new tests red before implementation, green after
- `grep -r "usage.prototype" src/ test/` returns nothing
- the suite spawns no subprocess: `bun test` passes with `claude` off `PATH`
- manually: `just web` with a run in flight — chips visible, panel opens,
  stays open across ≥10 SSE pushes, figures refresh underneath it
