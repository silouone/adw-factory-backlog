---
id: adw-cost-06-hop-on-context-size
type: bug
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A read-heavy session crosses 200k context long before the call cap hops it

> **Evidence, 2026-09-27, run `adw-usage-04-a-running-node-spends-in-the-dark-1790479260068`**
> (green, 108 min, 584 API calls, ~$26 at standard Sonnet rates). Per-session
> context, deduplicated by `message.id`: every build and test session stayed
> ≤157k, but both **plan** sessions ran to **265k and 242k** within their
> 60-call cap (~4k/call, read-heavy). That was **58 calls above 200k**, billed
> at the long-context premium. Same shape in `adw-perf-06-…-1790480156025`'s
> plan: 254k at 60 calls. `DEFAULT_NODE_TURN_CAP` bounds calls, not context.

## Requirements

- [x] **R1 — red first.** A `consumeAgentStream` test: a node with a context
      threshold of 150k streams assistant messages whose `usage` (input +
      cache_read + cache_creation) crosses 150k on the 5th distinct
      `message.id`, well under the turn cap. The node must end with the same
      `retry`/`TURN_CAP_MARKER` shape the turn cap produces, so `runCappedHops`
      hops. This fails on `main`.
- [x] **R2 — one trigger, two causes.** Context size is measured per distinct
      API call, from the message's own `usage`. Crossing
      `DEFAULT_NODE_CONTEXT_CAP` (**150_000** tokens; doc comment cites this
      evidence and the 200k premium boundary) triggers the existing hop path:
      checkpoint → artifact → synthesized (adw-bug-26/27). No new hop
      machinery.
- [x] **R3 — journaled.** The hop's journal event/reason says which bound
      fired (`turns` or `context`), with the observed context.
- [x] **R4 — Codex unaffected** when no usage is present on its messages; a
      message without usage never triggers.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`.
