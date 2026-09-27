---
id: adw-m8-02-engine-retry-override
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m1-05-engine]
attempts: []
---
# Engine: optional per-node retry target + ceiling

> Source of truth: `specs/adw-v1.1-lanes.md` (Impl. decision 10, §Engine). Read
> constitution + specs first (protocol §1). Prerequisite for the bug lane's
> second bounded loop.

## Context (grounded in source)

- `src/pipeline/engine.ts:120` — `LaneSpec = { nodes, repair, maxRounds, caps }`:
  a **single** repair target + **one** `maxRounds`.
- `engine.ts:416–453` — on a node's `retry`, the engine runs `lane.repair`, then
  re-runs the emitting node, up to `lane.maxRounds`. The round counter lives in
  the engine ("never in nodes", `engine.ts:165`).
- The bug lane needs **two** independent loops: red-check↺build-test-only (cap 2)
  and gates↺repair (cap 3). One shared target/ceiling can't express that.

## Requirements

- [ ] A node may declare an optional `retry?: { target: EngineNode; maxRounds:
      number }`. On a `retry` result the engine routes to `node.retry.target`
      (else `lane.repair`) and enforces `node.retry.maxRounds` (else
      `lane.maxRounds`); the counter stays in the engine.
- [ ] Journaling + spans bracket the override's target node exactly as they do
      `lane.repair` (Art. VI) — per-phase granularity preserved.
- [ ] **Backward-compatible:** a lane whose nodes declare no override behaves
      **byte-identically** — chore/feat proven unchanged.
- [ ] Ceilings remain declared in code (Art. V); a breach → stop, mark blocked,
      full transcript.

## Build protocol (red-first, Art. I)

1. Tests (`test/pipeline/engine.test.ts` prior art): a node with an override
   routes retries to its declared target and blocks at its own ceiling; a lane
   with no overrides is byte-identical (existing engine tests stay green). Confirm
   red, present for review.
2. Implement to green.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; override + fallback paths
covered; existing chore lane behavior unchanged.

## Out of scope

The red-check/base-green nodes (m8-05) and bug-lane wiring (m8-07) that consume
this.
