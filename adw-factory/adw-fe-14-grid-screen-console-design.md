---
id: adw-fe-14-grid-screen-console-design
type: feat
status: done
priority: 1
created: 2026-09-14
depends: [adw-bug-01-capture-parser-envelope]
attempts: [{"runId":"adw-fe-14-grid-screen-console-design-1789422369351","branch":"adw/adw-fe-14-grid-screen-console-design","workspace":"/Users/silouane/adw-factory/runs/adw-fe-14-grid-screen-console-design-1789422369351/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-14-grid-screen-console-design-1789647210372","branch":"adw/adw-fe-14-grid-screen-console-design","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-14-grid-screen-console-design-1789647210372/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/68","provider":"claude","model":"sonnet"}]
caps: {minutes: 150, turns: 700}
---
# The landing screen, rebuilt to variant D — a filterable board of ticket cards

> **Design is settled.** Operator direction across 2026-09-14, iterated live
> against a prototype. Build to **variant D**.
>
> **Reference implementation:** `src/web/render.prototype.ts` variant `D`, plus
> `src/web/GRID-PROTOTYPE-NOTES.md`. It runs: `just web`, then append
> `&variant=D` to the grid URL. `&variant=0` is today's grid for comparison.
>
> **Depends on `adw-bug-01`** and the dependency is real, not ceremonial: the
> activity dots in §4 render nothing until the capture parser is fixed, and a
> build that "implements" them against the current parser would go green
> having shipped an empty chart.

## 1. What is wrong with today's grid

`render.ts` predates the Console design that `adw-fe-13` landed on the run
screen. The two screens now encode different beliefs about what an operator is
reading — four ad-hoc hex literals vs a named token block, `system-ui` vs
`ui-monospace`, `<dt>state</dt><dd>finished</dd>` vs a status pill, and
`672431ms` printed as an integer.

And it does not scale: 114 banked runs render as 114 unbounded cards.

## 2. Structure

```
┌ HEADER   adw factory · N tickets · M runs   [All projects ▾] (GREEN)(BLOCKED)(NO RUN-END)
├ SECTION  ▸ adw-factory  23 tickets  ▬▬▬▬▬▬  65% green
│          └ a grid of TICKET cards
└ SECTION  ▸ unrecorded   (collapsed by default)
```

- [ ] **A card is a TICKET, not a run.** Attempts fold into dots on the card,
      each a link to its own run. 114 runs → 77 cards on today's corpus.
- [ ] **Sections are `<details>`** — native collapse, keyboard-accessible.
      Selecting a project from the dropdown auto-expands it.
- [ ] `unrecorded` is **not a peer project**. `target` is a recent journal
      field and the gap is backfillable; label it as legacy, sort it **last**,
      collapse it by default.

## 3. Filtering — and this is the part that takes scope from `adw-console-01`

- [ ] A project dropdown listing every target with its ticket count, plus
      "All projects". **No new data** — `run-start.target` has been journaled
      since `adw-fe-01`.
- [ ] **The stat pills ARE the health filter.** `GREEN` / `BLOCKED` /
      `NO RUN-END` are buttons, not a read-only strip.
- [ ] **Their counts recompute against the active project filter.** A count
      that ignored the filter would be the screen lying about its own state.
- [ ] Client-side, so it is instant and needs no new route.

> **Ledger note for the operator, not an instruction to this build:**
> `adw-console-01` (`status: blocked`) lists "filter and group runs by target"
> in its own scope. This ticket takes that slice. `adw-console-01` is
> deliberately **not amended here** — its banner exists because a prose-only
> constraint got overridden once, and editing a blocked ticket's scope in
> passing is the same mistake from the other side. Reconcile it by hand.

## 4. The card

Built from the operator's reference card. Every element below is **real
recorded data** — nothing on this card may be fabricated.

- [ ] **A mini-Gantt, one row per role**, bars positioned from real timestamps.
      `buildGanttView` is pure over journal records; reuse it (Art. VIII).
- [ ] **Activity dots, not solid spans.** A captured block draws its span as a
      dimmed rail with **a dot per tool call**, positioned by its own
      timestamp — `adw-fe-07`'s dots, at card scale.
- [ ] **A block with NO capture keeps the solid span.** This distinction is
      load-bearing: "never captured" and "made no tool calls" must not look
      identical. Same rule that gives the run screen its `cap-none` marker.
- [ ] **A step strip** beside the status tag — one dot per node that ran,
      coloured by its own outcome. Cap it, and **name the overflow** (`+3`);
      truncation is never silent.
- [ ] **KPI chips**: cost, duration, tokens. See §5.
- [ ] **The whole card is clickable** — stretched-link pattern (`::after` over
      the card from the real `<a>`), **not** an `<a>` wrapping the card: the
      attempt dots are links too and a nested `<a>` is invalid HTML that
      browsers silently un-nest. The dots sit above the overlay on z-index.
- [ ] **Uniform height.** Lane counts vary (chore 2, feat 4, bug 6); the chart
      reserves a fixed number of row slots and hides the unused ones, and the
      no-timeline fallback reserves the same height. The prototype measures
      264px for every card in a section — one distinct height.
- [ ] **No left accent bar.** Lane identity lives in the lane line and the
      outcome glyph.
- [ ] **A live run is unmistakable, and LIGHT.** One hue at low alpha: a cool
      ring + faint wash on the card, a `● live` pill with a slow beacon pulse,
      and `● N live` on the section rail so a **collapsed** section still says
      a run is happening inside it. A section holding a live run is never
      collapsed by default.
- [ ] **No size, padding or layout change when a card goes live.** The board is
      watched while it updates; a card that jumps on a state change costs the
      operator their place, and a layout-affecting mark reflows on every SSE
      swap. Colour and shadow only.
- [ ] `prefers-reduced-motion` replaces the pulse with a static halo — it must
      not remove the mark.

## 5. Cost and tokens — the two chips that must agree

This is the subtlest requirement on the screen and it has already been got
wrong once, inside the prototype, within one screen of adding a price.

- [ ] **The token chip shows TOTAL TOKENS PROCESSED** — input + output +
      cacheRead + cacheWrite — **not `billedTokens`.** Worked example from
      `adw-fe-13`'s own run: cache reads were 23,344,676 of 23,869,490 tokens
      (**98%**). Showing `billedTokens` (91K) beside a cache-inclusive price
      invites "$10 for 91K tokens", which is wrong by **260×**.
- [ ] `billedTokens` stays available in the tooltip, named as what
      `metrics.ts` reports. It is not hidden; it is just not the headline
      beside a price.
- [ ] **The cost chip is an estimate, derived and labelled.** `usage.model` and
      `usage.breakdown` are both journaled; only the rate is not. Price each
      block at its own model's rate, from the **full breakdown** — costing off
      `billedTokens` alone under-states by ~36×.
- [ ] **An unknown model yields NO estimate**, never a guessed rate. 54 of 77
      cards show `—` today (pre-`fe-01` runs with no `model`).
- [ ] **Always rendered `~`, with the rate and the date it was taken in the
      tooltip.**

> **Operator decision this build must surface, not settle:** where the rate
> card lives. A constant in a renderer goes stale silently. Options: a dated
> `pricing.json`, a field on `targets/*.json`, or the factory journaling a
> price at `node-end`. The prototype's inline constant is fine to look at and
> **not fine to ship** — if this is still undecided at build time, say so in
> the PR body rather than quietly shipping the constant.

## 6. Wording

- [ ] The non-agent lane is labelled **`code`**, not `workspace`. It is every
      node that spends no tokens — `dispatch`, `provision`, `assemble-*`,
      `baseline`, `gates`, `commit`, `push`, `open-pr` — and `code` names what
      they are, in the repo's own words: *"Agents propose, code disposes."*
- [ ] **Display only.** `WORKSPACE_LANE` stays the data key; `gantt.ts` owns it
      and the run screen reads it.
- [ ] The rail's tooltip carries the caveat: `gates` lives in this lane too,
      and a gate is the **target's** own command, not factory code.
- [ ] The free lane is dimmed, not lane-hued — Art. III made visible.

## 7. The SSE fragment boundary must move

`renderGridBody` is pushed verbatim by `/events` and swapped into
`#grid-body.innerHTML`. Every live-varying number this design adds — header
tallies, per-section meters, step dots — sits **above** today's boundary.

- [ ] Move the boundary so the header is inside the swapped fragment, and
      update `server.ts`'s `/events` route with it.
- [ ] **The active filter must survive a swap.** It lives in a closure in the
      prototype, so a tick would silently reset the board to "all". Re-apply
      after replace, or move the filter into the URL.
- [ ] `adw-fe-08`'s three render states (`running`/`finished`/`unknown`) are
      untouched, and a live run must still update in place.

## 8. Architecture

- [ ] **Renderers stay pure** — view object in, string out, no I/O
      (v1.2 decision 2). The prototype violates this with module-level mutable
      state and a `readFileSync`; **do not promote that**. Thread the data as
      parameters.
- [ ] **The per-run detail is a PROJECTION change, not a rendering one** —
      the biggest scope item here. `RunView` carries none of the card's chart,
      steps or tokens. The prototype gets them by reading each journal a
      second time and each capture file once: 114 runs, ~500ms, 683KB. Design
      the projection; do not ship three passes over the filesystem.
- [ ] `server.ts` stays a thin, **read-only** shell. No mutating route.
- [ ] **Zero new runtime dependencies** (v1.2 decision 11). The prototype is
      vanilla CSS + ~50 lines of JS and carries a dropdown, live filtering, a
      grouped board and a per-card chart — this answers
      `specs/adw-v1.5-operator-console.md` **D1** with **no**, for the third
      time.

## Verify

- [ ] Every card in a section renders at the same height — assert it, do not
      eyeball it.
- [ ] Filtering to a project hides every other section and recomputes the pill
      counts; toggling a pill hides those cards and the counts agree.
      (`[hidden]`'s UA `display:none` loses to any class rule that sets
      `display` — without an explicit guard the filter sets the attribute and
      **nothing visibly changes while the counts update correctly**. That is
      the worst failure mode available here; it was hit in the prototype.)
- [ ] A card with captures shows dots; a card without shows a solid span; they
      are distinguishable.
- [ ] Clicking anywhere on a card opens its run; clicking an attempt dot opens
      **that** attempt.
- [ ] Renderers unit-tested against hand-built view objects, no server.
- [ ] A live run still updates over `/events`, with a filter active.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## When done

- [ ] **Delete** `src/web/render.prototype.ts`,
      `src/web/GRID-PROTOTYPE-NOTES.md` and the `?variant=` branch in
      `server.ts`.
- [ ] **Do not promote prototype code.** It was written under prototype rules —
      no tests, module-level mutable state, I/O inside a renderer. Rewrite it
      properly into `render.ts`.

## Out of scope

The run screen — `adw-fe-15` aligns it. The dependency view and host RAM/CPU —
`adw-console-01`, still blocked for those.
