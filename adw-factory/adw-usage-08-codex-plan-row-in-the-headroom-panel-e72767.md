---
attempts: [{"runId":"adw-usage-08-codex-plan-row-in-the-headroom-panel-e72767-1790589259037","branch":"adw/adw-usage-08-codex-plan-row-in-the-headroom-panel-e72767","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-08-codex-plan-row-in-the-headroom-panel-e72767-1790589259037/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/144","provider":"claude","model":"sonnet"}]
id: adw-usage-08-codex-plan-row-in-the-headroom-panel-e72767
type: feat
status: in-review
priority: 2
created: 2026-09-28
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
---
# The headroom panel shows Codex's plan state, verbatim

> **Found 2026-09-28.** The headroom panel reads only `claude -p /usage`.
> Codex reports its limits in `rate_limits` events inside the rollout files
> under `~/.codex/sessions/YYYY/MM/DD/*.jsonl`, not in the `exec --json`
> stream the journal captures. On the GO1 business seat, all 332 records
> read:
> `{"limit_id":"codex","primary":null,"secondary":null,"credits":{"has_credits":true,"unlimited":true,"balance":null},"spend_control_reached":null,"plan_type":"business"}`.
> **There is no window to draw a gauge from. Do not invent one.**

## Requirements

- [ ] **R1 — red first: pure parser.** `parseCodexRateLimits(line)` takes
      a rollout line and returns `{plan, unlimited, balance, spendControlReached,
      primary?, secondary?}`, or `undefined` for non-matching lines. Test
      fixtures: the business/unlimited shape above; a shape with
      `primary: {used_percent, window_minutes, resets_at}` (a metered plan);
      and malformed JSON, which is skipped, not thrown.
- [ ] **R2 — injectable reader.** It finds the newest rollout file and
      returns its last `rate_limits`. It uses the same TTL-loader pattern as
      `createUsageLoader` (`src/web/usage.ts`), with the file system
      injected. No `codex` subprocess. If `~/.codex` is missing, it returns
      "no Codex data", never an error banner.
- [ ] **R3 — render (layout fixed, no prototype needed).** One row under the
      Claude gauges: `CODEX · business · unlimited credits · read Ns ago`.
      When `spend_control_reached` is true, the row turns red and reads
      *"GO1 spend control reached"*. When `primary` or `secondary` is
      present, render them with the existing `UsageGauge` component.
- [ ] **R4 — `/usage.json`** carries the Codex read alongside the Claude
      one, and the chip stays Claude-only.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web`: the
headroom panel shows the Codex row.
