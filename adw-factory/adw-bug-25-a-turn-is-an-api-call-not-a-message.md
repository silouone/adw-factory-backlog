---
id: adw-bug-25-a-turn-is-an-api-call-not-a-message
type: bug
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# The turn counter counts assistant messages, so every cap fires at half its budget

> **Evidence, 2026-09-27, run `adw-usage-04-a-running-node-spends-in-the-dark-1790472…`.**
> `build` blocked: `turn ceiling breached mid-stream: 52 turns observed, cap 51`.
> The run budget was 600. `plan` had "used" 347 across 5 hopped sessions.
> Transcript truth, top-level entries only: every session shows **~2.1 assistant
> entries per API call** (42 calls → 101 entries, 49 → 102, 46 → 102, 60 → 102,
> 57 → 102). One API call streams several `assistant` messages (thinking,
> text, each `tool_use`), all with the same `message.id`.
> This over-count was latent (the M1 watch item: "mid-stream turn cap counts SDK
> assistant messages") until `adw-bug-23` made top-level turns count at all.
> Now the node cap (100) hops at ~50 calls, and the run ceiling blocks at ~half
> the ticket's `caps.turns`.

## Requirements

- [x] **R1 — red first.** A `consumeAgentStream` test feeds 3 API calls, each
      streamed as 3 top-level `assistant` messages sharing one `message.id`.
      `usage.turns` must be 3, and a node cap of 3 must not fire until the 4th
      distinct id. This fails on `main`.
- [x] **R2 — count distinct API calls.** A top-level turn is a distinct
      `message.id` (`parent_tool_use_id == null`). Messages with no id (the Codex
      path, some fixtures) each count as one turn, as today. Subagent messages
      still don't count.
- [x] **R3 — every consumer agrees.** The node-local cap, the run-level
      mid-stream ceiling, the reported `usage.turns`, and the hop's
      `turnsFromPriorHops` all use the same count. Grep for other counters of
      assistant messages (turn batching, web metrics) and state in the PR
      which ones mean "API call".
- [x] **R4 — re-derive the node cap.** With true calls, measured growth is
      ~1.5k tokens per call (cost-04 build: 29.1k → 104.5k over 51 calls), so
      100 calls ≈ 180k, too close to the 200k long-context premium. Set
      `DEFAULT_NODE_TURN_CAP` to **60** (≈ 120k at the hop). Say so in its doc
      comment, with this evidence.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then re-dispatch `adw-usage-04`.

## Blocked by

None.
