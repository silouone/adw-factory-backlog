---
id: adw-m8-05-red-check-nodes
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m1-08-gates-node]
attempts: []
---
# `classifyRed` + run-all `red-check` node + `base-green-check` node

> Source of truth: `specs/adw-v1.1-lanes.md` (Graph C; Impl. decisions 7, 8, 9,
> 13). Read constitution + specs first. This is the core new deterministic logic
> — the ONE new pure seam.

## Context (grounded in source)

- `src/pipeline/nodes/gates.ts` — runs `target.gates` in declared order, **stops
  at first fail** ("Later gates are never run", S2.1). Reuse its gate *execution*;
  the classifier needs COMPLETE results, so red-check runs a **run-all** variant.
- `src/targets/loader.ts:16` — `Gate = { name, cmd }`, ordered. cLens: `lint`,
  `typecheck`, `test`.
- Base is not proven green anywhere today (provision runs `setup` only) —
  base-green-check is new. Art. VII governs push, not the cut base.

## Requirements

- [ ] **Pure `classifyRed(orderedGateResults, testGateName) → "clean-red" |
      "test-passed" | "broken-test"`:** `clean-red` iff the `test`-gate result is
      a **failure** AND every non-test result is a **pass**; `test-passed` iff the
      `test` gate passed; `broken-test` iff any non-test gate failed. Pure — no
      I/O, no clock.
- [ ] **`red-check` node**: runs **all** gates unconditionally, collecting every
      result (NOT stop-at-first — else a target whose `test` gate isn't last
      yields a false `clean-red`), calls `classifyRed`, exposes the classification
      + a precise reason (revise-loop / block wiring is m8-07).
- [ ] **`base-green-check` node**: runs the gate suite on the untouched checkout;
      all green → `next`; any red → a **block** outcome naming the failing gate
      ("target base not in a provable-clean state", Art. IX). May reuse
      stop-at-first (first red is enough to block).
- [ ] Errors carry ticket id + node name (Art. IX).

## Build protocol (red-first, Art. I)

1. Tests: `classifyRed` all three branches with **fabricated** ordered
   gate-result arrays (incl. non-test red while test red → `broken-test`; test
   red + others green → `clean-red`; test green → `test-passed`). Node tests via
   a fake `workspace.exec`: red-check runs ALL gates even past a failure;
   base-green `next`/`block`. Prior art: `test/pipeline/nodes/gates.test.ts`.
   Confirm red, present for review.
2. Implement to green.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; every classifier branch +
both nodes covered; run-all collection asserted; zero I/O in classifier tests.

## Out of scope

The bounded revise loop, lane composition, fail-fast precondition (m8-07).
