---
id: adw-m8-10-lane-wrapper-consolidation
type: chore
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m8-06-feat-lane, adw-m8-07-bug-lane]
attempts: [{"runId":"adw-m8-10-lane-wrapper-consolidation-1789347958327","branch":"adw/adw-m8-10-lane-wrapper-consolidation","workspace":"/Users/silouane/adw-factory/runs/adw-m8-10-lane-wrapper-consolidation-1789347958327/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/30","provider":"claude","model":"sonnet"},{"runId":"adw-m8-10-lane-wrapper-consolidation-1789598712680","branch":"adw/adw-m8-10-lane-wrapper-consolidation-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-m8-10-lane-wrapper-consolidation-1789598712680/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/60","provider":"claude","model":"sonnet"}]
---
# Consolidate the triplicated lane wrapper nodes (chore/feat/bug)

> Follow-up from the M8 two-axis code-review (Standards axis, judgement call #1).
> NOT a blocker — the three lanes shipped green + validated; this is a craft
> consolidation to honor Art. VIII ("one representation", "a wrapper once two
> call sites demand one") now that the threshold is exceeded at n=3 lanes.

## Context (grounded in source)

With `feat.ts` (m8-06) and `bug.ts` (m8-07) landed, three lane modules now carry
verbatim/near-verbatim copies:

- `dispatchNode`, `provisionNode`, and the `required<T>(ctx, key)` helper are
  triplicated across `src/pipeline/lanes/{chore,feat,bug}.ts` (only the deps type
  name differs).
- `feat.ts` `assembleStageNode` and `bug.ts` `assembleNode` are near-twins
  (`threadsPlan` vs `withPlan`).
- `ChoreLaneDeps` / `FeatLaneDeps` / `BugLaneDeps` share ~9 identical fields (a
  Data Clump); `FEAT_MAX_ROUNDS`/`BUG_MAX_ROUNDS` + `FEAT_CAPS`/`BUG_CAPS` copy
  chore's module-private `CHORE_*` values — `feat.ts` itself flags the drift risk.

Art. VIII permits a wrapper once "two call sites demand one"; that threshold is
now exceeded, so the documented standard argues *for* extraction.

## Requirements

- [ ] Extract the shared deterministic wrapper nodes (`dispatchNode`,
      `provisionNode`, `required`, the assemble-stage node) into one shared module
      the three lanes import — no behavior change.
- [ ] Give the shared lane deps their one representation (a base `LaneDeps` the
      three specialize), so the ~9 common fields live once.
- [ ] Export the lane caps/rounds defaults (`CHORE_MAX_ROUNDS`/`CHORE_CAPS`) so
      feat/bug reference them instead of copying the literals (kill the drift risk
      feat.ts documents).
- [ ] Behavior-preserving: chore snapshot byte-identical; all three lane test
      suites green before AND after (Art. I refactor discipline — no new red).

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; chore snapshot untouched;
the three lane suites + cli selection tests unchanged and green.

## Out of scope

Any behavior change to the lanes; the live bars (m8-08/09).
