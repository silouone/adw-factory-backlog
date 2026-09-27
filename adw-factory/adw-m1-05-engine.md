---
id: adw-m1-05-engine
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-04-journal]
attempts: []
---
# Blueprint engine

## Context

The heart of the pipeline (plan §3): a lane is an ordered `Node[]`; the
engine owns everything nodes must not — retry accounting, ceilings, journal
middleware, abort. Nodes stay pure logic. Bounded autonomy is structural:
"no unbounded retry exists anywhere" (Art. V).

## Deliverables

- `test/pipeline/engine.test.ts` (first, red — fake nodes, fake clock)
- `src/pipeline/engine.ts`

## Requirements

- [x] Types: `Node = { name, run(ctx) → NodeResult }`;
      `NodeResult = next(ctxPatch?) | retry(payload) | fail(reason)`;
      immutable ctx, strict TS (plan §3, Art. IX)
- [x] Sequential execution of the lane's node list; `retry` routes to the
      lane-declared repair node then re-runs the retried node; engine owns
      the round counter (S2.2 shape)
- [x] Round ceiling from lane config (chore: 3): the 4th failure of the
      retried node → outcome `blocked` with the full failure history attached
      (S2.3)
- [x] Wall-clock + turn ceilings read from lane config, overridable per
      ticket via `caps:`; breach → graceful abort (S2.5)
- [x] Graceful abort (ceiling, SIGINT via AbortController, or injected
      abort): outcome `blocked`, `abort` + `run-end` journal events flushed,
      workspace **kept** — teardown is never called on abort paths (S2.6,
      S4.4, Art. VII)
- [x] Journal middleware wraps every node: `node-start`/`node-end` emitted by
      the engine, not by nodes — a node cannot run unjournaled (Art. VI)
- [x] Node errors are wrapped to carry `ticketId` + node name (Art. IX)

Note (operator-approved API, 2026-07-14): `usage` rides `next`/`retry`
results so the engine owns turn accounting and journals agent usage —
nodes never track budgets. `blocked` is the single terminal failure
outcome, distinguished by `reason`. Wall-clock is checked at node
boundaries; mid-flight agent aborts are m1-09's job via the distributed
`AbortSignal`. Engine emits `run-start`/`run-end` itself (isolation kind
via opts, default worktree).

## Build protocol (Art. I)

1. Tests with stub nodes: happy path ordering; retry→repair→re-run cycle;
   blocked on 4th failure with history; per-ticket cap override honored;
   fake-clock ceiling breach → abort semantics; SIGINT abort; journal
   middleware coverage (every node run has matched start/end events); error
   wrapping.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

Concrete nodes (m1-07…09), lane definition (m1-10), spans (M3).
