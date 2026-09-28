---
attempts: [{"runId":"adw-usage-09-spend-by-provider-bc23c8-1790611089112","branch":"adw/adw-usage-09-spend-by-provider-bc23c8","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-09-spend-by-provider-bc23c8-1790611089112/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
id: adw-usage-09-spend-by-provider-bc23c8
type: feat
status: blocked
priority: 2
created: 2026-09-28
caps: {minutes: 180, turns: 400, stallMinutes: 25}
depends: [adw-usage-06-a-codex-run-is-journaled-as-claude-128a3d, adw-usage-07-price-codex-runs-and-journal-the-rate-713388, adw-usage-08-codex-plan-row-in-the-headroom-panel-e72767]
---
# Spend by provider: Claude vs Codex, 24h and 7d

> **The operator's ask, 2026-09-28:** "we don't have vision on the
> consumption". The Codex seat is enterprise and unmetered (usage-08), so
> the only consumption view is the factory's own journals. Build that view.

## Requirements

- [ ] **R1 — red first: pure rollup.** `spendByProvider(blocks, now)`
      returns, per provider and per window (last 24h, last 7d): tokens, the
      breakdown, run count and `estimateUsd`. It keys on
      `profile.provider` (fixed in usage-06). Blocks with no provider go in
      an explicit `unknown` bucket and are never dropped.
- [ ] **R2 — honest money.** The Claude ≈$ figure is labelled *"API-equivalent,
      you pay a subscription"* and the Codex figure *"API-equivalent, not
      the GO1 bill"*. An unpriced share is shown as `N tok unpriced`, never
      as $0.
- [ ] **R3 — render.** A `FACTORY SPEND BY PROVIDER` table in the headroom
      panel, under usage-08's Codex row. Its columns match the existing
      LAST 24H and LAST 7D columns.
- [ ] **R4 — scope note.** Label it "factory runs only". Interactive Claude or
      Codex sessions are not in the journals.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`: the
table shows both providers, with cmc's Codex runs priced.
