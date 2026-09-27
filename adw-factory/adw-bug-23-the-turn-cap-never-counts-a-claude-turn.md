---
id: adw-bug-23-the-turn-cap-never-counts-a-claude-turn
type: bug
status: done
priority: 1
created: 2026-09-26
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: []
---
# The node turn cap never counts a Claude turn, so sessions grow to 600k tokens and never hop

> **Evidence, 2026-09-26, run `adw-store-01-tickets-dir-1790448708493`.** The
> build node's prompt says "This session is capped at 100 turns". The node
> made **473 API calls in one session** and never hopped. Context grew from
> 27k to **598k tokens** per call, averaging 401k. The session consumed 189.6 M
> input tokens (99.6% cache reads) and cost **~$63 at standard Sonnet rates,
> ~$121 if the >200k long-context premium applies**. A session hopping every
> 100 turns, under ~100k context, would have cost roughly $18.

## Root cause (to confirm with a red test first)

The mid-stream counter in `src/pipeline/nodes/build.ts` counts a turn only when:

```ts
message.type === "assistant" &&
message.parent_tool_use_id === undefined // subagent turns don't count
```

The Agent SDK types this field `parent_tool_use_id: string | null`
(`sdk.d.ts`). The factory's local mirror declares `parent_tool_use_id?: string`.
For a top-level Claude message the value is `null`, so `null === undefined` is
false and **no Claude turn is ever counted**. Consequences:

- The node-local cap (`DEFAULT_NODE_TURN_CAP = 100`, `adw-cost-02`) never
  fires, so the fresh-session-from-checkpoint hop never happens.
- The run-level mid-stream ceiling (`ctx.caps.turns`) never fires either.
  Across every banked journal, `breached mid-stream` appears **0 times**. The
  only "RESUMED SESSION (hop" strings on disk come from `adw-cost-02`'s own
  build run.
- The Codex path is unaffected: `codex-query.ts` sets `undefined` itself.

## Requirements

- [x] **R1 — red first.** A `consumeAgentStream` test that feeds Claude-shaped
      assistant messages with `parent_tool_use_id: null` and a node cap of 3
      asserts `TURN_CAP_MARKER` on the 4th message. It must fail on `main`.
- [x] **R2 — fix the predicate** so that `null` and `undefined` both mean
      "top-level" (`== null`), and align the local message type with the SDK
      (`string | null`). Subagent messages (non-null id) still don't count.
- [x] **R3 — every other site.** Grep for every `parent_tool_use_id ===
      undefined` or `!== undefined` comparison in `src/` (turn batching,
      watchdog, capture, web) and apply the same null-safe rule, each with a
      test.
- [x] **R4 — prove the hop.** A `runCappedHops` test with null-parented
      messages crosses the cap, reads the checkpoint, and starts a fresh
      session without `resume`.
- [x] **R5 — re-derive the constants.** State in the PR body what the
      100-turn node cap now means in practice: at about 1.1 API calls per
      turn, a hop comes every ~100 calls, well under 200k context. Say
      whether the cap should be lower for `build`.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`. Then one live feat run:
the journal shows a hop at turn 101, and the second session's first call
context is back near the initial prompt size.

## Blocked by

None. This ticket can start immediately.
