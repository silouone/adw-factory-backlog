---
id: adw-route-04-an-ab-report-for-one-ticket-d9fec2
type: feat
status: queued
priority: 1
created: 2026-10-03
depends: [adw-route-01-a-gear-ladder-and-honest-prices-476b13]
attempts: []
---
# Compare two runs of the same ticket on turns, tokens, cost, wall-clock and outcome

> Spec: `specs/adw-v1.16-model-routing.md` D6, discharging `adw-cost-02` R3/R3a (left unchecked on a done ticket).

## Context

The base routing table is a hypothesis. Its build, test and repair cells must be checked by running the same ticket on two gears (for example G2 vs G1). Today there's no way to put two runs side by side. `run-autopsy` and `run-scorecard` each read one run.

## Requirements

- [ ] **R1:** a script takes two run ids, or two groups of run ids, for the same ticket. It prints one table with:
  - per node: gear, model, reasoning, turns, total tokens (input / output / cache-read / cache-write), API-equivalent cost from the rate card, and wall-clock;
  - per run: outcome, repair rounds and total wall-clock.
- [ ] **R2:** a column of deltas (B vs A) per node and per run.
- [ ] **R3:** it's built on the existing journal readers and the rate card. No new journal parsing.
- [ ] **R4:** runs on different tickets are refused with a clear error. An unpriced model shows as unpriced, never a guessed rate.
- [ ] **R5:** `just ab <runA> <runB>` recipe.

## Operator step after merge (not part of this ticket)

Run the A/B (feat G2 vs G1; chore G0 vs G0+) and record the result in the spec's routing table.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
