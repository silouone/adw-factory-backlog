---
attempts: [{"runId":"adw-usage-07-price-codex-runs-and-journal-the-rate-713388-1790578758323","branch":"adw/adw-usage-07-price-codex-runs-and-journal-the-rate-713388","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-07-price-codex-runs-and-journal-the-rate-713388-1790578758323/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/142","provider":"claude","model":"sonnet"}]
id: adw-usage-07-price-codex-runs-and-journal-the-rate-713388
type: feat
status: done
priority: 1
created: 2026-09-28
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: [adw-usage-06-a-codex-run-is-journaled-as-claude-128a3d]
---
# Price Codex runs, and journal the rate a node ran at

> **Operator decision 2026-09-28.** This resolves the open question in
> `src/web/pricing.ts`'s header with option **(c): the factory journals the
> price at `node-end`**. Reason: the gpt-5.6-sol rate is a promo, valid at
> least through 2026-11-21. A rate card read at render time would silently
> reprice every old run when it expires. **This changes the journal schema,
> so propose the spec amendment (`specs/adw-v1-plan.md`) in the PR first,
> per the amendment rule.**
>
> Today a Codex card shows `$ —`: `USD_PER_MTOK` keys only
> `opus|sonnet|haiku`, and `rateFor` is a loose `includes()`, first match
> wins.

**Rates (source: developers.openai.com/api/docs/models/gpt-5.6-sol, read
2026-09-28), USD per MTok:** `gpt-5.6-sol`: input 4, cacheRead 0.4,
cacheWrite 5, output 20. Prompts over 272K input are 2× input and 1.5×
output for the whole request. The journal keeps totals, not per-request
sizes, so this tier cannot be applied. The basis string must say the
estimate under-counts long-context turns. Price no other gpt slug until its
rate is read from a primary source (rule 2: unknown means no estimate).

**Semantics already verified (no work needed):** `reasoning_output_tokens`
is inside `output_tokens`. `cache_write_input_tokens` was 0 in all 332 rollout
records. `parseUsage` already splits fresh and cached input (#124).

## Requirements

- [ ] **R1 — red first: matching.** Tests for `rateFor`: `gpt-5.6-sol`
      gets the sol rate. `gpt-5.6-terra`, `gpt-6-sol`, `gpt-6-astra` and
      `codex-auto-review` get **no** rate. `claude-opus-*`, `sonnet` and
      `haiku` keep today's rates. Keys are exact slugs or ordered
      most-specific-first. No bare `gpt` family key.
- [ ] **R2 — the rate card moves out of `src/web/`** into a factory-side
      module that both the pipeline and the web import. `src/web/` must not
      become a dependency of `src/pipeline/`.
- [ ] **R3 — journal the rate.** A `node-end` usage record carries the
      resolved `rate` (the four fields plus `asOf`, and a `promoUntil` where
      one applies), or omits it when the model is unpriced. Red test on the
      node-end builder first.
- [ ] **R4 — price from the journal first.** `estimateUsd` prices a block
      from its journaled `rate`. It falls back to the card for older
      journals without one, and the basis says which it used. Live blocks
      (`liveUsage`) use the card.
- [ ] **R5 — honest basis for gpt.** When any gpt block is priced, the
      tooltip says: *"API list price, not your enterprise bill"*, the promo
      date, and the >272K under-count. Test the exact string.
- [ ] **R6 — the cmc card.** On a banked Codex journal, the card renders
      `~$N` instead of `$ —`. Put before/after figures in the PR.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` and
check that a Codex run card shows `~$…` with the enterprise caveat on hover.
