---
id: adw-route-07-size-and-extreme-in-shadow-71446a
type: feat
status: queued
priority: 2
created: 2026-10-03
depends: [adw-route-04-an-ab-report-for-one-ticket-d9fec2, adw-route-05-the-router-plans-every-node-6d2cdb]
attempts: []
---
# The router reads ticket size and a declared extreme, in shadow until the data says go

> Spec: `specs/adw-v1.16-model-routing.md`: the small, large and extreme columns, D3, D5, D7.

## Requirements

- [ ] **R1: size** is a pure function of the ticket file:
  - number of `- [ ]` requirement lines;
  - body length;
  - distinct repo paths named;
  - declared `caps`.

  It returns `small | normal | large`. The thresholds are named constants. The initial proposal (small: ≤ 3 requirements and < 2k chars; large: ≥ 10 requirements, or > 8k chars, or caps.turns > 600) is documented in the doc comment as provisional.
- [ ] **R2: `complexity: extreme`** is an optional frontmatter field. Any other value is refused at parse time.
- [ ] **R3:** the router applies the spec's small, large and extreme columns. Extreme puts plan and review-spec on G5 in feat and bug, and review-spec on G5 in chore.
- [ ] **R4: size and extreme decisions are shadow-only** until a per-target `routingSize: live` is set, independent of `routing`. The `route` event records the size, the inputs that produced it, and the would-be gears.
- [ ] **R5:** route-04's report gains a shadow mode. Across N runs it compares, per node, shadow gear vs the gear that ran, against outcome, turns and wall-clock, so the go-live decision has data.

## Operator step

Go live on size and extreme after about 30 shadow dispatches across both providers.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
