---
attempts: [{"runId":"adw-usage-11-three-token-pills-bfa3dc-1790593179391","branch":"adw/adw-usage-11-three-token-pills-bfa3dc","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-11-three-token-pills-bfa3dc-1790593179391/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/145","provider":"claude","model":"sonnet"}]
id: adw-usage-11-three-token-pills-bfa3dc
type: bug
status: in-progress
priority: 1
created: 2026-09-28
caps: {minutes: 90, turns: 300, stallMinutes: 20}
depends: []
---
# The board card lost its total-tokens figure when it gained an input/output split

> **Found 2026-09-28, verified against a live journal.** `adw-usage-10`
> (#141) replaced the board card's single `◈` KPI — which used to render
> `summary.totalTokens` (input+output+cacheRead+cacheWrite, `card.ts`'s own
> doc-comment: "the TOTAL TOKENS PROCESSED figure §5 requires as the token
> chip's headline") — with an input/output-only split. Verified on
> `runs/adw-usage-10-split-input-output-tokens-560d93-1790580470326/journal.jsonl`:
> `input: 444, output: 1913, cacheRead: 14989329, cacheWrite: 492011`. The
> card now shows `444 in / 2K out` next to `$6.37` — the ~15M cache tokens
> that actually drove that price vanished from the card, with nowhere else
> on it to see them. (The run screen's `RunHeader.tsx` is unaffected — it
> kept its separate `context` chip alongside the new input/output split.)
>
> **The operator's fix, 2026-09-28:** three pills, board card only.
>
> 1. The existing input/output pill, unchanged (`444 in / 2K out`).
> 2. A new pill, immediately next to it: cache read/write (e.g. `15.0M
>    read / 492K write`).
> 3. A new, last pill: the overall total — restoring `summary.totalTokens`,
>    the figure that actually matches the `$` chip.

## Requirements

- [ ] **R1 — red first: `cacheWriteTokens` on `CardSummary`.** `card.ts`
      already hand-sums `cacheReadTokens` from `usage.breakdown.cacheRead`
      (falling back to `liveUsage.breakdown.cacheRead`) because no existing
      `NodeMetrics` field exposes it alone (`card.ts:93-99, 191-194`). Add
      `cacheWriteTokens` the same way, from `usage.breakdown.cacheWrite` /
      `liveUsage.breakdown.cacheWrite`. A unit test proves the sum over a
      multi-block view, mirroring the existing `cacheReadTokens` test.
- [ ] **R2 — three pills on the board card, `Card.tsx`.** Replace the single
      `kpi-tok` chip with three, in this order, right after the `◷` time
      chip:
      1. unchanged: the input/output split (`fmtTokenSplit`, as shipped in
         #141) — `summary.inputTokens` / `summary.outputTokens`.
      2. new: cache read/write, same `in X / out Y`-shaped formatting,
         sourced from `summary.cacheReadTokens` / R1's `cacheWriteTokens`.
         Pick a distinct glyph, consistent with the existing single-Unicode-
         symbol convention (`$`, `◷`, `◈`).
      3. new, last: `summary.totalTokens` (already computed, currently
         unrendered — no new derivation needed), formatted with the
         existing `fmtTokenCount` helper. This is the figure that should
         read consistently against the `$` chip's basis.
      All three carry `~` while `summary.live` is true, same convention as
      today. `tokenTooltip` keeps its input/output/cache-read lines and
      gains a cache-write line.
- [ ] **R3 — scope: board card only.** `RunHeader.tsx` (the run screen) is
      unchanged — it already shows `input`, `output` and `context` (the
      cache-inclusive total) as separate chips; this ticket does not touch
      it.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`: a
board card shows three separate token pills — input/output, cache
read/write, and a total that reads consistently next to the `$` estimate.
