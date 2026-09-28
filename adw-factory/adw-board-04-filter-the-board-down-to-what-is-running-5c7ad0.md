---
id: adw-board-04-filter-the-board-down-to-what-is-running-5c7ad0
type: feat
status: done
priority: 1
created: 2026-09-28
caps: {minutes: 90, turns: 400, stallMinutes: 20}
depends: []
attempts: [{"runId":"adw-board-04-filter-the-board-down-to-what-is-running-5c7ad0-1790610034294","branch":"adw/adw-board-04-filter-the-board-down-to-what-is-running-5c7ad0","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-board-04-filter-the-board-down-to-what-is-running-5c7ad0-1790610034294/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/150","provider":"claude","model":"sonnet"}]
---
# The board cannot be filtered down to the runs that are live right now

> **Operator, 2026-09-28.** The header filter row is
> `[1D][3D][7D][ALL] histogram · All projects ▾ · GREEN 17 · BLOCKED 7 ·
> NO RUN-END 7`. With a wave in flight there is no way to say *"show me
> only what is building right now"* — the live cards are scattered through
> 42 finished ones, and `NO RUN-END` is the wrong proxy: it also holds
> every stalled and every crashed run.
>
> **The ask:** a LIVE filter in that row, next to the three health chips.

## LIVE is not a fourth health bucket — and that is the whole design

`Health` is a **strict three-way partition** of a card by its latest
attempt's outcome (`board-groups.ts`), and `visibleCardsFor` filters with a
lookup keyed on it: `health[healthOf(c.latest)]`. A live card is *also*
`unfinished` — `CardTally.live` says so in its own doc-comment: *"Not one
of the three filter buckets — a live run always also counts as
`unfinished`."*

So LIVE **must not** become a fourth `Health` key. It is an intersecting
toggle, ANDed on top of the existing partition: off (default) the board is
byte-identical to today; on, only `isLive(card.latest)` cards survive,
regardless of the health chips.

## Requirements

- [ ] **R1 — red first: the signal, `board-filter.ts` /
      `test/web/ui/board-filter.test.ts`.** A fourth module-level signal,
      `liveOnly = signal<boolean>(false)`, beside `proj`/`health`/
      `menuOpen`. Same writer discipline as its three siblings, stated in
      the module comment: **`Header.tsx` is the only writer**; every other
      component reads it as a prop handed down by `Board.tsx`.
- [ ] **R2 — red first: the filter arithmetic, `board-groups.ts` /
      `test/web/ui/board-groups.test.ts`.** `visibleCardsFor` gains a
      fourth parameter `liveOnly: boolean`, ANDed after the health lookup.
      Tests: `liveOnly:false` is byte-identical to today's result for the
      existing fixtures (pin this explicitly — it is the no-regression
      claim); `liveOnly:true` with mixed cards returns only
      `state:"running"` ones; `liveOnly:true` **with every health chip
      off** returns nothing (AND, never OR); `liveOnly:true` with no live
      card returns `[]`.
- [ ] **R3 — the chip, `Header.tsx` / `test/web/ui/Header.test.tsx`.** A
      `LIVE` chip in `.gh-r`, rendered **after** the three health chips,
      riding the existing `.gh-r .chip` rules with a new `live` variant
      (`.gh-r .chip.live.on{color:var(--live);border-color:…}` in
      `css.ts`, using the `--live` theme var that already exists — no new
      token). Its count is `visibleTally.live`, which `CardTally` already
      computes and which the header already receives. Clicking it toggles
      `liveOnly`. When the count is `0` the chip still renders (reading
      `LIVE 0` is the answer to "is anything running?"), and clicking it
      legitimately empties the board.
- [ ] **R4 — every downstream site, or it ships inconsistent.** Three
      places recompute filtered counts independently and all three must
      honour `liveOnly`, each with its own assertion:
      - `Board.tsx` / `BoardHeader` — passes `liveOnly.value` into
        `visibleCardsFor` for both `visibleTally` and `visibleTicketCount`,
        and hands it down to `Section` as a plain prop (never imported by
        `Section` itself — same relationship `proj`/`health` already have);
      - `Board.tsx`'s `.fx-empty` guard ("No ticket matches the active
        filter") — must appear when `liveOnly` alone empties the board;
      - `Section.tsx` — its own `visibleCards` / `visibleTally` (the meter,
        the shown-ticket count, the `% green` rate, the `.grp-empty` note)
        recompute against `liveOnly` too. `liveTally` (the unfiltered
        `● N live` badge and the mount-time default-open rule) is
        **unchanged** — it is deliberately filter-proof, per that file's
        own header comment.
- [ ] **R5 — not persisted.** `liveOnly` is session state, like
      `proj`/`health`. No URL param, no `replaceState`, no localStorage.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` with a
run in flight: clicking `LIVE` leaves only the running cards, the section
meters and counts follow, clicking it again restores the board exactly.
