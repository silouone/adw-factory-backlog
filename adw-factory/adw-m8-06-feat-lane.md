---
id: adw-m8-06-feat-lane
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m8-03-plan-test-agent-nodes, adw-m8-04-prompt-templates]
attempts: []
---
# `featLane()` — Planner-led feature graph + orchestrator selection

> Source of truth: `specs/adw-v1.1-lanes.md` (Graph B; Impl. decisions 1, 15).
> Read constitution + specs first.

## Context (grounded in source)

- `src/pipeline/engine.ts:121` — the engine iterates `lane.nodes`; a new lane =
  a different ordered node list.
- `src/pipeline/lanes/chore.ts:115` — `choreLane(deps)` composes Graph A;
  `CHORE_CAPS`, `CHORE_MAX_ROUNDS`, per-ticket overrides. Mirror its structure.
- `src/pipeline/nodes/build.ts`, m8-03's `plan`/`test` nodes — the agent stages.
- `feat` uses only the **existing `gates ↻ repair`** loop (Art. III — the gate is
  the failure signal); NO per-node retry override needed here.

## Requirements

- [ ] **`featLane(deps)`** returns a `LaneSpec` with the node list:
      `dispatch → provision → assemble(plan) → plan → assemble(build) →
      build(tests-first) → assemble(test) → test → gates → commit → push →
      open-pr`, with the existing `repair` node on the gates tail (cap 3).
- [ ] The `plan` artifact threads into the build prompt (`{{plan}}`, m8-03).
- [ ] Build uses `feature-build.md` (tests-first); test uses `feature-test.md`
      (coverage); plan uses `feature-plan.md`.
- [ ] **Orchestrator selection:** `ticket.type === "feat"` → `featLane`; a code
      switch (no router agent).
- [ ] Reuses chore caps to start (decision 15); per-ticket `caps:`/`model:`
      overrides unchanged.

## Build protocol (red-first, Art. I)

1. Tests (faked `AgentQuery`, prior art `test/pipeline/*`): the engine running
   `featLane` executes plan→build→test→gates in order; the plan artifact reaches
   the build prompt; a red gate routes to `repair` (existing loop); type→lane
   selection picks `featLane` for `feat`. Confirm red, present for review.
2. Implement to green — chore path proven untouched.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; the plan→build→test flow,
plan threading, gates↻repair, and lane selection covered; chore unchanged.

## Out of scope

The live proofs (m8-08/09) — offline-green wiring only (offline-green is NOT
sufficient; the live bar is separate, the M7 lesson).
