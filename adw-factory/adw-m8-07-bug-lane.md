---
id: adw-m8-07-bug-lane
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends:
  - adw-m8-02-engine-retry-override
  - adw-m8-03-plan-test-agent-nodes
  - adw-m8-04-prompt-templates
  - adw-m8-05-red-check-nodes
attempts: []
---
# `bugLane()` — Plan-led, two-phase machine-enforced red-first graph

> Source of truth: `specs/adw-v1.1-lanes.md` (Graph C; Impl. decisions 7–13).
> Read constitution + specs first. This ticket wires the rigor showcase.

## Context (grounded in source)

- `src/pipeline/engine.ts` — the per-node `retry?:{target,maxRounds}` override
  (m8-02) powers the red-check loop; `LaneSpec` + `lane.nodes` iteration.
- `src/pipeline/nodes/build.ts` — `AgentQueryOptions.resume` (build.ts:82); the
  `repair` node's same-session resume (S2.2). Turns count per stream; minutes are
  run-wide.
- m8-03 `plan` node + `{{plan}}`; m8-05 `red-check` + `base-green-check` nodes.
- `src/targets/loader.ts` — `Gate[]`; the `test`-gate convention (Graph C).

## Requirements

- [ ] **`bugLane(deps)`** returns a `LaneSpec` with the node list:
      `dispatch → provision → base-green-check → assemble(plan) → plan →
      assemble(test-only) → build(test-only) → red-check → assemble(fix) →
      build(fix·resume) → gates → commit → push → open-pr`, plus `repair` on the
      gates tail (cap 3).
- [ ] **red-check bounded revise loop, cap 2** via the per-node retry override
      (`retry: { target: build-test-only, maxRounds: 2 }`): non-`clean-red` →
      re-run the test-only build with the precise reason; exhaustion → **block**.
- [ ] **build(fix) resumes** the test-only session (thread its `sessionId` as
      `resume`); uses `bug-build-fix.md`. Phase 1 uses `bug-build-test.md`; plan
      uses `bug-plan.md` and threads `{{plan}}` into phase 1.
- [ ] **base-green red → block** before the plan agent runs (no tokens on a dirty
      base).
- [ ] **Fail-fast precondition:** a `bug` whose target has no gate named `test`,
      or where `test` is the only gate, is **rejected before provisioning** with a
      descriptive error carrying the ticket id (Art. IX).
- [ ] **Orchestrator selection:** `ticket.type === "bug"` → `bugLane` (code
      switch). Reuses chore caps to start (decision 15).

## Build protocol (red-first, Art. I)

1. Tests (faked `AgentQuery`): the engine running `bugLane` gates the fix on
   `clean-red`; a `test-passed`/`broken-test` drives the revise loop and **blocks
   after 2**; build(fix) carries the phase-1 `sessionId`; base-green red → block
   before plan; type→lane selection; fail-fast rejection for a target without a
   separable `test` gate. Confirm red, present for review.
2. Implement to green — chore/feat lanes proven untouched.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; two-phase flow, revise-cap
block, resume threading, base-green block, lane selection, fail-fast all covered.

## Implementation notes

- **Revise target (clarification).** The graph shorthand above writes the
  red-check retry as `↻ revise → build test-only`. It is implemented as a
  **distinct repair-shaped node** (`revise-test-only`), *not* the plain
  `build-test-only` build node. A plain `makeBuildNode` reads `ctx.data.prompt`
  (the original test prompt) and **ignores `ctx.repairPayload`**, so the agent
  would never receive the `classifyRed` reason — story 7 ("tell the agent
  *exactly why*") would be silently dead, and offline-green would still pass
  (the M7 masking trap). The revise node mirrors `repair.ts`: it **resumes** the
  test-only session and formats the precise reason into its prompt (Art. IX /
  story 7). Like `lane.repair`, it is `red-check.retry.target` and **not** a
  member of the `nodes` list.

## Out of scope

The live proofs (m8-08/09). Offline-green is NOT sufficient (M7 lesson).
