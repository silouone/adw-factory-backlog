---
id: adw-bug-35-flaky-baseline-passes-are-cached-as-a-red-base-6688ec
type: bug
status: in-progress
priority: 1
created: 2026-10-01
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A baseline whose passes fail on different tests is cached as a red base, and every run on that commit is refused

Measured 2026-10-01 on base `6b058da` (adw-factory). Three runs were refused at `baseline-green-check`:
- `adw-target-01-silou-hq-is-a-target`
- `adw-web-01-a-stable-address`
- `adw-web-02-a-one-shot-board-snapshot`

All three gave the same reason: *"gate test is already failing on the base commit … (the failing
test names could not be read)"*. Each one lost ~35 minutes before being refused.

The cached baseline (`runs/baselines/6b058da….json`) shows:
- **2 passes, both exit 1, 17 failed of 5207 each time, but the names did not repeat.**
- `reproducing: []`, so all 17 (rebase tests) were recorded under `flaky`.
- `unattributable: true`, which is what blocked the runs.

`reconcileBaselineRuns` (introduced by adw-bug-31, #166) says *"a test gate that exited non-zero in
every pass is red whether or not a name could be read"*. That rule was meant for **unreadable**
names (a missing or unparsed report). It now also fires when names **were read in every pass and
none reproduced**, which is the factory's own definition of flaky.

The same base was checked on a clean worktree: both rebase files pass 69/69 in isolation (see
adw-bug-36 for why they fail under load). The block reason is also wrong: the names could be read.

## The fix

- **Red, unattributable** = every pass failed **and at least one pass's failing names could not be read**.
- Every pass failed with readable names and an **empty intersection** = flaky, **not** pre-existing. The gate resolves `pass` for the base, and the flake is recorded.
- Optional: a third pass is the tie-break. A reproduction in it makes the gate red.
- The block reason must say which case applied: unreadable, reproducing (with names), or flaky (never blocks).

## Red first (Art. I)

- `reconcileBaselineRuns` with two failing passes and disjoint readable names gives the test gate `pass`, `flaky` holding the union, and `unattributable: false`.
- Two failing passes where one has no readable names gives `fail`, `unattributable: true` (adw-bug-31's case, still pinned).
- Two failing passes sharing one name gives `fail`, with `failingTests` = that name.

## Acceptance criteria

- [ ] The three cases above are tests, and adw-bug-31's existing tests still pass.
- [ ] A spec amendment if adw-bug-31's wording in the spec says "non-zero every pass ⇒ red" (Amendment rule).
- [ ] The cache schema is bumped, so verdicts cached under the old rule are recomputed.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
