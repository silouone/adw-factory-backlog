---
id: adw-auto-01-baseline-gate-snapshot
type: feat
status: done
priority: 1
created: 2026-09-12
depends: []
caps: {minutes: 120, turns: 600}
attempts: [{"runId":"adw-auto-01-baseline-gate-snapshot-1789218434781","branch":"adw/adw-auto-01-baseline-gate-snapshot","workspace":"/Users/silouane/adw-factory/runs/adw-auto-01-baseline-gate-snapshot-1789218434781/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Know which gates already fail on the base, so an agent never chases a failure it did not cause

> Minted 2026-09-12 from the `adw-fe-01-journal-schema` incident. Not a
> timing problem and not fixable by tuning — an autonomy gap.

## What happened

Run `adw-fe-01-journal-schema-1789213958548` spent **47 minutes in its `test`
node** in this loop:

```
bun test (full, ~7 min) → check one file (5 s) → bun test (full, ~7 min) → …
```

The suite reported **920 pass / 4 skip / 1 fail**, and the single failure was
`provisionContainer — auth slots (adw-m4-03)` — a Docker test that spins real
containers.

**The agent did not cause it.** Its diff touched journal, engine, build and
cli; the failing test exercises `container.ts`, untouched. The same suite ran
**0 fail** earlier the same day on the same machine. The failure is
environmental — the suite spins **12 concurrent containers** and one test
loses under its own contention.

The agent could not know any of that. From inside the node, one test fails, so
it retries. It would have looped until the 120-minute wall clock destroyed the
run — and a wall-clock breach aborts everything, so `adw-caps-01`'s
deterministic-tail fix does **not** rescue it.

**The operator diagnosed it by hand in ten minutes, by knowing the suite had
passed earlier. That knowledge is what the factory is missing.** Its own run
logs keep making the same comparison by hand — `adw-m5-06`: *"the 2–4
pre-existing real-subprocess timeouts fail identically on main."*

## The fix — subtract the baseline

Record what already fails on the **unmodified base tree**, then subtract it
from what fails after the agent's work. A failure present in both is not the
agent's problem and must never charge a repair round.

## Requirements

- [ ] After `provision` and **before any agent node**, run the target's gates
      once against the **unmodified base tree** and record the result: per
      gate pass/fail, and for a test gate the **set of failing test names**.
- [ ] Journal it as a `baseline` event, so a run's own history answers "what
      was already broken."
- [ ] **⚠ A single baseline sample is NOT sufficient — the base is
      nondeterministic.** Measured 2026-09-12 on one unchanged tree, alone,
      with a freshly cleaned Docker: **2 failures, then 6**. Main measured
      **0** in the same conditions. Three of the six were real-subprocess
      timeouts at exactly ~5000 ms; three were container-lane tests. A
      one-shot baseline would have recorded "0 fail" and then charged the
      agent for all six — **precisely the wrong conclusion a human reached
      from the same one-shot evidence.**
      The baseline must therefore be a **set union over N samples** (start at
      2, named constant), or carry an explicit **quarantine list** of
      known-nondeterministic tests. Either way the journal records how many
      samples produced it, so a thin baseline is visible rather than trusted.
- [ ] **Cache by base commit sha** (`runs/baselines/<sha>.json`). The baseline
      for a given base commit is stable, and most runs share one — without
      this the factory pays a full gate cycle per run, which for this repo is
      ~7 minutes. A cache hit must be journaled as such, so a stale baseline is
      never invisible.
- [ ] At `gates`, partition failures into **pre-existing** (∩ baseline) and
      **introduced** (∖ baseline). Only *introduced* failures route to repair
      or fail the run.
- [ ] A run that is green except for pre-existing failures **passes**, and the
      journal plus the PR body name the pre-existing set explicitly. Silence
      here would be the same lie as counting them: the operator must see that
      the base is dirty.
- [ ] The agent's prompt is told the pre-existing set, so it does not chase
      them from inside the node either. **This is the part that actually stops
      the loop** — subtraction at the gate saves the run, but only telling the
      agent saves the 47 minutes.
- [ ] A baseline gate that is itself unrunnable (docker down, missing venv) is
      journaled and the run proceeds with an **empty** baseline. A sick
      baseline must never block a run (Art. VI).

## Verify

- [ ] Given a baseline with failure X, a post-agent run failing only X →
      gates **pass**, zero repair rounds charged.
- [ ] Baseline with X, run fails X **and** Y → only Y routes to repair, and the
      repair payload names only Y.
- [ ] A cache hit for an unchanged base sha skips the baseline run and says so
      in the journal.
- [ ] An unrunnable baseline yields an empty baseline, a journaled reason, and
      a run that proceeds.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Fixing the flaky container test (`adw-flaky-01`). Deciding whether a dirty base
should block dispatch — it should be *visible* first.

## Run log

**2026-09-12 — run `…-1789218434781` BLOCKED: liveness watchdog on `build`,
"no SDK stream message for 10 minutes".**

**Root cause was CONTENTION, not this ticket's code.** It ran fully concurrent
with `adw-flaky-01-container-tests-contention` (22 seconds apart, both
container-heavy) for 66 minutes. The last completed command was
`bun test test/pipeline/nodes/ci-round.test.ts` taking **1124 s — 18.7 minutes
for one file**. The watchdog fired at the 10-minute mark while that legitimate
test was still running.

Work verified and salvaged: 27 files, +1327/−56. `bun run lint` clean (88
files), `bunx tsc --noEmit` clean. Both load-bearing pieces are present —
`{{preExisting}}` in the build prompts (the requirement that actually stops the
loop) and `introducedFailingTests` in the gates node (the subtraction).
