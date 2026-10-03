---
id: adw-route-06-a-struggling-stage-climbs-a-gear-62e8d9
type: feat
status: queued
priority: 2
created: 2026-10-03
depends: [adw-route-05-the-router-plans-every-node-6d2cdb]
attempts: []
---
# A struggling stage climbs one gear, inside its existing round budget and on its next dispatch

> Spec: `specs/adw-v1.16-model-routing.md` D9, routing table *esc* column.

## Context

- **A blocked run is the factory's most expensive failure.** Today the last repair round runs on the same gear as the first.
- **A re-dispatched blocked ticket repeats the same gear** on the stage that failed.

## Requirements

- [ ] **R1: in-run.** The router's plan carries a gear per round for each bounded-loop node:
  - repair: rounds 1–2 at the planned gear, round 3 one gear up;
  - revise-test-only: round 2 one gear up;
  - review-fix: round 2 one gear up.

  The node reads its gear by the engine's round number. No node counts rounds, and **no round is added** (D9).
- [ ] **R2: cross-run.** If the ticket's last attempt has `blockedAt.node = X` and a capability-shaped reason, node X starts one gear up. An extreme ticket's build-family node escalates to G5.
- [ ] **R3: capability-shaped is a pure classifier** over the recorded block reason. These **never** escalate:
  - approval or classifier outage;
  - provision failure;
  - red base or baseline refusal;
  - dispatch refusal;
  - stall from an external abort.

  These do:
  - exhausted repair, revise or review-fix rounds;
  - a node `fail` from the agent's own work;
  - a turn or context ceiling on a node.
- [ ] **R4:** escalation only raises a gear, is clamped at G5, never changes a level-1 ticket pin, and journals `{node, round, from, to, why}`.
- [ ] **R5:** in `shadow` mode escalation is journaled as `shadowGear` only.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
