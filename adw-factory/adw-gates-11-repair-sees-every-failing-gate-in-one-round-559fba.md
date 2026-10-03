---
id: adw-gates-11-repair-sees-every-failing-gate-in-one-round-559fba
type: feat
status: done
priority: 2
created: 2026-10-02
caps: {minutes: 150, turns: 400}
depends: []
attempts: [{"runId":"adw-gates-11-repair-sees-every-failing-gate-in-one-round-559fba-1790989666352","branch":"adw/adw-gates-11-repair-sees-every-failing-gate-in-one-round-559fba","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-11-repair-sees-every-failing-gate-in-one-round-559fba-1790989666352/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/186","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Repair sees every failing gate in one round, not one gate per round

> **Refine at pickup.** Decision D1 below needs an operator pick before this goes `in-progress`.

## Why

The gates node stops at the first introduced failure (`src/pipeline/nodes/gates.ts` ~:222, "S2.1 stop-at-first-INTRODUCED"). Repair therefore sees one gate's failure per round, and finds the next gate's failures only by burning another round.

**hq-07, run `…-1790925359587`:**
- All 4 gate passes failed at `lint`.
- `typecheck` never ran, and the tree had **118 tsc errors** in the new tests.
- Even a repair that cleared lint in round 3 would have reached a 4th gate pass and died at tsc with no rounds left. The 3-round budget could not have covered the work, and nothing showed that until the hand salvage.
- **hq-05:** rounds swung between lint and tsc, each round fixing the one it could see.

## What to build

After the first introduced failure, the remaining gates still run, **report-only**:
1. Their results go into the gate summary and the repair prompt, each marked `after-failure`. The first failure stays the headline.
2. They never change routing. The run goes to repair because of the first failure, exactly as today.
3. A later gate's failure is classified against the baseline as usual: pre-existing vs introduced.
4. The repair prompt lists every introduced failure. Each gate gets its own share of the excerpt budget, so one noisy gate cannot push out the others.

**D1 — what a report-only test gate costs.** adw-factory's `test` gate runs for minutes, and adw-gates-02 exists to stop paying for the suite repeatedly. Options:
- **(a) Recommended:** run every remaining gate report-only, except a gate that declares `reportOnlyAfterFailure: false`. adw-factory's `test` declares it; silou-hq, clens and sabado don't need to.
- (b) Run only the gates declared *before* the test gate. Simpler, but it hides test failures behind a lint failure.
- (c) A target-level switch.

## Spec amendment

Amend S2.1 from "stop at first introduced" to "route on the first introduced failure; report every gate". Record D1's outcome and the cost rationale next to adw-gates-02.

## Red first (Art. I)

Gates node, with a fake workspace:
- lint fails and typecheck fails → both are in the summary, typecheck marked `after-failure`, route = repair, and the repair prompt contains both excerpts;
- lint fails and typecheck passes → the route and the first-failure reason are byte-identical to today's;
- a gate opted out per D1 is not executed after a failure, and is recorded as skipped with its reason (Art. V, never dropped silently);
- a later gate that fails on the baseline too is recorded `pre-existing`, not introduced.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`
- Replay check: build the repair prompt from hq-07's round-1 workspace state. It must name both the Biome errors and the tsc errors.
