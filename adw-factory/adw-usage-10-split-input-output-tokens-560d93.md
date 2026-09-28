---
attempts: [{"runId":"adw-usage-10-split-input-output-tokens-560d93-1790580470326","branch":"adw/adw-usage-10-split-input-output-tokens-560d93","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-10-split-input-output-tokens-560d93-1790580470326/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/141","provider":"claude","model":"sonnet"}]
id: adw-usage-10-split-input-output-tokens-560d93
type: feat
status: in-review
priority: 2
created: 2026-09-28
caps: {minutes: 90, turns: 300, stallMinutes: 20}
depends: []
---
# Split the token chip into INPUT / OUTPUT, on the card and the run header

> **The operator's ask, 2026-09-28**, looking at a board card showing
> `$ ~$13.0`, `⏱ 83m 00s`, `◈ 33.08M`: model prices are rarely ISO between
> input and output (`src/web/pricing.ts` prices `sonnet` at `{input: 3,
> output: 15}` — a 5x spread, same ratio for `opus` and `haiku`), but the
> token chip next to the price is a single flat number. Split it so the
> figure that actually drives the $ estimate is visible.

## Requirements

- [ ] **R1 — red first: the metrics split.** `attributeBlockMetrics`
      (`src/web/metrics.ts`) already reads `usage.breakdown.input` /
      `.output` to build `contextTokens` (input+output+cacheRead+cacheWrite)
      but never exposes the input/output split alone. Add
      `inputTokens`/`outputTokens` to `NodeMetrics`, derived the same way
      `contextTokens` is: `undefined` when `usage.breakdown` is absent
      (pre-fe-01 journal), never a fabricated `0`. Cover the live-node path
      (`attributeBlockLiveMetrics`) too, from `liveUsage.breakdown`. A unit
      test proves both the finished and live cases.
- [ ] **R2 — card rollup.** `CardSummary` (`src/web/card.ts`) sums
      `inputTokens`/`outputTokens` across every block the same way
      `cacheReadTokens` already is (`card.ts:183-186`): read
      `b.usage?.breakdown?.input`/`.output`, falling back to
      `b.liveUsage?.breakdown.input`/`.output`, defaulting to `0` only at
      the sum, never per-block.
- [ ] **R3 — the board card.** `Card.tsx`'s `kpi-tok` chip (the `◈` KPI,
      currently `fmtTokens(summary.totalTokens, summary.live)`) renders the
      input/output split instead of one flat figure — e.g. `12.4M in /
      20.7M out`. `tokenTooltip` keeps its existing billed/cache-read lines,
      with input and output stated separately rather than folded into
      `billedTokens`.
- [ ] **R4 — the run header.** `RunHeader.tsx`'s `billed` chip (currently
      one `fmtNum(billed)` summed from `block.metrics?.billedTokens`) splits
      into an input figure and an output figure, sourced from R1's new
      `NodeMetrics` fields.
- [ ] **R5 — no new pricing logic.** `pricing.ts` already prices `rate.input`
      and `rate.output` separately; this ticket only surfaces the split
      that already drives `estimateUsd`. Do not touch `estimateUsd` or add a
      second $ figure.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`: the
board card's token chip and the run screen's header chip each show separate
input/output figures instead of one flat token count.
