---
id: adw-usage-05-a-multi-session-node-reports-only-its-last-session
type: bug
status: done
priority: 2
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: [adw-usage-04-a-running-node-spends-in-the-dark]
attempts: [{"runId":"adw-usage-05-a-multi-session-node-reports-only-its-last-session-1790488468589","branch":"adw/adw-usage-05-a-multi-session-node-reports-only-its-last-session","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-05-a-multi-session-node-reports-only-its-last-session-1790488468589/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/124","provider":"claude","model":"sonnet"}]
---
# A node that hopped sessions reports only its last session's tokens at `node-end`

> **Found 2026-09-27 while auditing `adw-usage-04` (PR #121).** When a capped
> node hops (turn cap, context cap, or poisoned session), its `node-end` usage
> sums **turns** across sessions, but `tokens` and the `breakdown` (input,
> output, cacheRead, cacheWrite) come from the **last** session only. The live
> figure (usage-04, which carries across hops) can therefore be *higher* than
> the final figure, and every §6 cost metric under-reports any node that
> hopped. With `DEFAULT_NODE_TURN_CAP = 60` and the context cap
> (`adw-cost-06`), most build nodes now hop, so this is the common case.
> Related gap found in the same audit: Codex `parseUsage` (`codex-query.ts`)
> drops `cached_input_tokens`, so Codex cache reads show as 0.

## Requirements

- [x] **R1 — red first.** A `runCappedHops` test with two sessions of known
      usage: `node-end` usage equals the **sum** of both sessions for tokens and
      every breakdown field. This fails on `main`.
- [x] **R2 — sum across sessions**, using the same accumulation the live
      figure uses, so the live figure and the final figure agree when the node
      ends.
- [x] **R3 — Codex cache reads.** `parseUsage` maps `cached_input_tokens`
      to `cacheRead`, with a test from the captured Codex fixture.
- [x] **R4 — metrics.** `bun scripts/run-metrics.ts` totals are unchanged for
      single-session nodes. State this in the PR with before/after on a banked
      journal.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`.
