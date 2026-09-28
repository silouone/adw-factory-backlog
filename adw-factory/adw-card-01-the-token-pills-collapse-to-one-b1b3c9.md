---
id: adw-card-01-the-token-pills-collapse-to-one-b1b3c9
type: chore
status: in-review
priority: 1
created: 2026-09-28
caps: {minutes: 60, turns: 200, stallMinutes: 15}
depends: []
attempts: [{"runId":"adw-card-01-the-token-pills-collapse-to-one-b1b3c9-1790609130207","branch":"adw/adw-card-01-the-token-pills-collapse-to-one-b1b3c9","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-card-01-the-token-pills-collapse-to-one-b1b3c9-1790609130207/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/148","provider":"claude","model":"sonnet"}]
---
# The board card's token row reads as noise — collapse three pills into one

> **Operator, 2026-09-28, looking at a live board.** The card's KPI row now
> wraps onto three lines: `$ ~$3.42` · `◷ 67m 55s` / `◈ 322 in / 1K out` /
> `⇄ 6.84M read / 359K write` · `Σ 7.20M`. Token consumption takes more
> vertical space than the mini-Gantt it sits under, and none of those five
> numbers is the one being scanned for.
>
> **What the card is actually scanned for:** the price and the total. Every
> other figure is a drill-down.

This **deliberately narrows** `adw-usage-10` (#141) and `adw-usage-11`
(#145, merged today). It is not a regression report against either: both
put the right figures on the card, and this ticket keeps every one of
them — it moves the breakdown from four always-visible pills to the hover
tooltip that **already carries all of it**.

`tokenTooltip` (`Card.tsx`) today emits, on every one of the three token
pills: total processed · cache reads + % · input · output · cache writes ·
turns · model. Nothing is lost by hiding the pills.

## Requirements

- [ ] **R1 — red first, `test/web/ui/Card.test.tsx`.** Adapt the existing
      `kpi-tok` / `kpi-tok-cache` / `kpi-tok-total` assertions into their
      post-change form and confirm them red on `main`:
      - a rendered card contains **no** `.kpi-tok` and no `.kpi-tok-cache`
        element;
      - it still contains `.kpi-cost`, `.kpi-time` and `.kpi-tok-total`;
      - `.kpi-tok-total`'s `title` still carries the full breakdown —
        input, output, cache read, cache write, total, turns — i.e. the
        same `tokenTooltip` string, unchanged.
- [ ] **R2 — drop the two pills, `src/web/ui/Card.tsx`.** Remove the
      `kpi-tok` (input/output) and `kpi-tok-cache` (cache read/write)
      `<Kpi/>` calls. The surviving row is exactly three pills, in this
      order: `$` cost · `◷` time · `Σ` total tokens. `tokenTooltip` itself
      is **unchanged** — same lines, same order.
- [ ] **R3 — clean up only what R2 orphaned.** `fmtTokenSplit` and
      `fmtCacheSplit` lose their only call sites; delete them and their
      doc-comments. `fmtTokenCount`, `fmtTotalTokens` and `tokenTooltip`
      all stay. Nothing else in the file moves.
- [ ] **R4 — the dead CSS goes with them, `src/web/ui/css.ts`.** Remove the
      `.kpi-tok` and `.kpi-tok-cache` colour rules (css.ts:338-339);
      `.kpi-tok-total` stays. If `test/web/ui/css.test.ts` asserts on
      either selector, update that assertion in the same red step as R1.
- [ ] **R5 — scope: the board card only.** `RunHeader.tsx` (the run
      screen) keeps its separate `input` / `output` / `context` chips
      untouched — the run screen is the drill-down, and it is where the
      split belongs. No projection, no `card.ts`, no wire change:
      `CardSummary` keeps every field it has today, `inputTokens` /
      `outputTokens` / `cacheReadTokens` / `cacheWriteTokens` included —
      they still feed the tooltip.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`: a
board card's KPI row fits one line — price, wall time, total tokens — and
hovering the `Σ` pill still reveals input, output, cache read, cache
write, turns and model.
