---
id: adw-route-05-the-router-plans-every-node-6d2cdb
type: feat
status: queued
priority: 1
created: 2026-10-03
depends: [adw-route-02-every-agent-node-is-routable-5d55f8, adw-route-03-codex-runs-a-gear-per-stage-a1f747]
caps: {minutes: 180, turns: 900}
attempts: []
---
# The router plans every agent node's gear from one table, and every target can adopt it

> Spec: `specs/adw-v1.16-model-routing.md`: the routing table (normal column), D5, D7, D11, and *Implementation Decisions → Router*.
> **Operator step first:** amendment v1.16 is approved and committed before this is dispatched (it lifts v1.8 §8).

## Context

route-01..03 make every agent node *able* to run any gear. Nothing *chooses* one: every node falls through to G2. This ticket adds the chooser as a pure, deterministic function.

## Requirements

- [ ] **R1: the router.** A pure, total function from (lane, ticket, target, provider) to a per-node plan `{node → {gear, model, reasoning, reasons[]}}` covering every agent node of the lane, plus `ci-repair` and `rebase-resolve`. The base is the spec's **normal** column, built in.
- [ ] **R2: how it composes with the existing chain.**
  - A ticket per-stage pin is absolute.
  - A ticket whole-ticket `agent:`/`model:` replaces the base for all nodes.
  - Target `agents` replaces built-in cells.
  - The built-in table fills the rest.
- [ ] **R3: floors.** In feat and bug, plan and both reviews never resolve below G3 unless pinned by a level-1 ticket setting, and build-family nodes never below G0+. Values are clamped to G0..G5.
- [ ] **R4: mode per target.** `routing: off | shadow | live`, default `off`.
  - `off`: byte-identical to today.
  - `shadow`: plan computed and journaled, nodes run today's resolution.
  - `live`: nodes run the plan.
- [ ] **R5: journal.** A `route` event at dispatch carrying the full plan, the mode, and the inputs. Each agent node-end carries `gear` and, in shadow, `shadowGear`.
- [ ] **R6: surfaces.** The run view shows a gear chip per agent node. The PR body lists each node's gear and model.
- [ ] **R7:** an unknown `routing` value is refused at target load.

## Out of scope

Size, extreme (route-07), escalation (route-06).

## Operator step after merge

Set targets to `shadow`, and to `live` once the route-04 A/B confirms the base table.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
