---
id: adw-m1-08-gates-node
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-06-worktree-workspace, adw-m1-07-target-config-prompt]
attempts: []
---
# Gates node — lint → typecheck → test, as code

## Context

"Never an agent where a function suffices" (Art. III): gates run as plain
subprocess calls through `workspace.exec`, strictly ordered, zero agent
involvement (S2.1). Their structured output feeds both the repair loop and
the PR gate table.

## Deliverables

- `test/pipeline/nodes/gates.test.ts` (first, red — fake Workspace)
- `src/pipeline/nodes/gates.ts`

## Requirements

- [x] Run `target.gates` in declared order via `workspace.exec`; stop at the
      first failure (S2.1)
- [x] Failure → `retry(GateFailureReport)` where the report carries: gate
      name, command, exit code, stdout/stderr tails, and the workspace's
      diff-so-far (`git diff` vs base) to anchor the repair agent
      (S2.2, plan §6 thrash mitigation)
- [x] Command that cannot execute (spawn error, missing script) is a
      **target-configuration error**, not a gate failure: `fail(blocked)`
      typed as config error, **no repair round charged** (E3)
- [x] All gates green → `next` with per-gate results (name, duration, pass)
      for the journal and later the PR table (S1.5, S5.1)

## Build protocol (Art. I)

1. Fake Workspace whose `exec` is scripted per command. Tests: ordering;
   stop-at-first-failure; report shape incl. diff-so-far; exec-error vs
   failure distinction; all-green result shape.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

Repair itself (adw-m1-09), CI checks (adw-m2-04).
