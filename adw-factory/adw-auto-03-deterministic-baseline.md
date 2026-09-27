---
id: adw-auto-03-deterministic-baseline
type: feat
status: done
priority: 2
created: 2026-09-13
depends: [adw-flaky-01-container-tests-contention]
attempts: [{"runId":"adw-auto-03-deterministic-baseline-1789515243292","branch":"adw/adw-auto-03-deterministic-baseline","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-auto-03-deterministic-baseline-1789515243292/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/58","provider":"claude","model":"sonnet"}]
---
# A baseline snapshotted from a flaky suite launders the flake in both directions

> Minted 2026-09-13. Follow-up to the operator decision recorded in
> `ai_docs/2026-09-13-amendment-proposal-preexisting-gates.md` (option B landed;
> this is option A). Deliberately blocked on `adw-flaky-01` — it is worth very
> little against a nondeterministic suite.

## Evidence

`adw-auto-01`'s `baseline` node runs the target's gates on the base tree and
records what fails, so the agent is never told to chase a failure it did not
cause. The test gate compares by parsed `(fail) <name>` lines —
`introduced = current − baseline` — which is the careful part of the design.

It is stacked on a suite that returned **0, 1, 2, 3 and 6 failures across five
runs on three trees of the same code**, every failure a container or
real-subprocess test, never a logic test.

- The snapshot captures whichever tests flaked **at that instant**.
- A later gate run flaking on the **same** names → `introduced = []` →
  `fail-preexisting` → the failure is hidden from the agent and from repair.
- A later run flaking on **different** names → `introduced ≠ []` → the agent is
  told to fix a failure it did not cause — **the exact outcome this ticket's
  parent exists to prevent**.
- The baseline is **cached by base commit sha** (`runs/baselines/<sha>.json`),
  so one unlucky snapshot poisons every subsequent run on that commit.

`runs/baselines/` did not exist on the operator machine as of 2026-09-13 — no
run had exercised it yet. The hazard is live but not yet realised.

## Requirements

- [ ] Classify a failure `pre-existing` only when it **reproduces**: run the
      base gates more than once and take the intersection. A failure that
      appears in one base run and not another is a flake, not a baseline.
- [ ] The cache must record how many base runs it is built from, so a
      single-run baseline from before this change is not silently trusted.
- [ ] A flaky-but-not-reproducing base failure must be **visible** — journaled
      and surfaced — not silently dropped into either bucket.
- [ ] Measure the added cost against the sha cache before choosing the number
      of base runs.

## Verify

- A unit test over the pure classification: the same failing-test name present
  in base run 1 and absent in base run 2 is NOT pre-existing.
- The baseline cache from a prior single-run snapshot is not reused as if it
  were a reproducing one.
