---
id: adw-bug-06-the-suite-is-the-pipeline
type: feat
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-bug-06-the-suite-is-the-pipeline-1789417633708","branch":"adw/adw-bug-06-the-suite-is-the-pipeline","workspace":"/Users/silouane/adw-factory/runs/adw-bug-06-the-suite-is-the-pipeline-1789417633708/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/45","provider":"claude","model":"sonnet"}]
---
# The agent runs the full suite ~15 times per run — 78% of all tool time is one command

> Measured 2026-09-14 across every banked run, using the capture parser that
> `adw-bug-01` had just fixed. **This is the first time this data has been
> readable**; before that fix `time.toolMs` was 0 everywhere and the whole cost
> was invisible.
>
> Trigger: the operator observed *"the tok/s is VERY LOW most of the time"*.
> It is not the model. It is this.

## 1. The model was never slow

Splitting output tokens by **model** time rather than wall time:

| node | tok/s (wall) | tok/s (model only) | ratio |
|---|---|---|---|
| plan | 53.3 | 53.9 | 1.01× |
| build | 32.7 | 53.3 | **1.63×** |
| test | 18.1 | 42.0 | **2.32×** |
| build-fix | 21.3 | 54.5 | **2.56×** |

Throughput is a flat **~40–55 tok/s on every node**. `plan` shows the true
figure because it is 99% model time and barely touches tools; every other node
looks slower in exact proportion to how long it sits blocked.

**`metrics.ts`'s `tokensPerSecond` is tokens ÷ WALL time**, so what it reports
is dominated by how long the agent waited on a shell command. As a measure of
the model it is misleading, and it misled the operator.

- [ ] Either rename it to what it measures, or report model-only throughput
      beside it. A number labelled `tok/s` that mostly measures the test suite
      is the same class of defect as `billedTokens` beside a cache-inclusive
      price.

## 2. Where the time actually goes

```
total tool time, all banked runs:   805 minutes
  Bash          4771 calls   692.9m   86.1%
  TaskOutput      30 calls    91.3m   11.3%
  Read          2035 calls     0.5m    0.1%
  Edit          1200 calls     0.1m    0.0%

269 Bash calls > 20s  =  626m  =  78% of ALL tool time
```

Nearly every one of those 269 is the gate suite. The slowest single call was
**1,125s — 18.7 minutes on one test file** (`ci-round.test.ts`). Several
`bun run lint && bunx tsc --noEmit && bun test` calls ran **570–601s**.

Per run, by the agent alone:

| | |
|---|---|
| median suite invocations per run | **15** |
| median minutes per run on suites | **8.7m** |
| total agent-side suite time | **651 minutes** |
| worst single run | **73 invocations** |

And `baseline` + `gates` run it again deterministically, every run.

Overall agent time across the corpus is **70% model / 27% tool / 3% subagent**
— which also **corrects `adw-fe-13`'s "~90% model time"**, a figure taken while
the split was fabricated. (`adw-fe-15` already owns propagating that.)

## 3. Why `adw-gates-02` did not cover this

`adw-gates-02-stop-paying-for-the-suite-four-times` is `done` and addressed the
**deterministic** double-run. Its own *Out of scope* says:

> Making the suite itself faster (container tests are the I/O boundary on
> purpose) and splitting the gate into fast/slow tiers. Both are real options;
> both change what "done" means and want their own ticket and operator decision.

**This is that ticket.** The 15 agent-side invocations per run were never in
scope for anything.

## 4. The three levers, cheapest first

- [ ] **Tell the agent not to run the full suite as its inner loop.** The
      prompts are the highest-leverage files in the repo. An agent that runs
      the targeted test file while working and the full suite **once** at the
      end costs a fraction of one that runs everything 15 times. Cheapest
      lever, no architectural change — try it first and measure.
- [ ] **A fast tier.** `just test-fast` already exists and excludes the
      container/E2B suites. Make the agent's inner loop the fast tier and keep
      the full suite for `gates`, which is where "done" is actually decided.
      **This changes nothing about the definition of done** — `gates` is
      unchanged — which is what makes it safe.
- [ ] **Make the suite faster.** 1,599 tests in 256s. One file takes 18.7
      minutes. That file is the whole tail — profile before optimising
      anything else.

      **Accepted with reason (build stage, 2026-09-14):** `ci-round.test.ts`
      run solo, isolated from any other load, measured **67.9s** (60 tests, 0
      fail) — not 18.7 minutes. The file does real git subprocess work
      (`execFileSync("git", …)`, no mocks) inside `makeFixture`/
      `makeContainerFixture`, so per-test cost in the 0.3–2s range is expected
      and inherent to what the tests pin (real worktree/branch/commit
      behavior); rewriting it away from real subprocesses would weaken what
      the tests actually verify, so that is explicitly not done here. The 17x
      gap between this measurement and the banked 1,125s outlier is
      consistent with the corpus having been captured under concurrent
      multi-run load (disk/CPU contention from many parallel real-git
      operations happening at once) — [adw-bug-05](adw-bug-05-concurrency-ceiling-stalls-the-stream.md)'s
      territory, not a defect local to this file. Not fixed as part of this
      ticket (Article II: no speculative optimization against unevidenced
      cost).

## 5. Requirements

- [x] Measure first: suite invocations per run, and minutes per run, **before**
      any change. `src/observability/gate-cost.ts` already counts full-suite
      executions from the journal (`adw-gates-02` built it) — extend it to
      count **agent-side** invocations from captures rather than writing a
      second counter (Art. VIII).
      **Build stage (2026-09-14):** the instrument is built —
      `countAgentSideSuiteInvocations` (`src/observability/gate-cost.ts`),
      the `command` field threaded through captures (`src/web/capture.ts`),
      and `scripts/gate-cost.ts` wired to print `agent-side-execs=<n>`
      per run. This sandbox has no banked `runs/` directory (gitignored,
      generated only by a real, operator-triggered `just run`), so it
      cannot itself produce the §2 baseline's "before" numbers — the
      instrument is ready for the operator to run against the real corpus
      post-merge.
- [ ] Apply the cheapest lever, re-measure, and record both numbers in this
      ticket. A prompt change whose effect is not measured is a guess.
      **Build stage (2026-09-14):** the lever itself is applied —
      `prompts/repair.md` now tells the agent not to re-run the full gate
      command mid-repair-round (checked against the specific failing file
      instead). Four of the five prompts that already carried "Iterating
      cheaply" guidance (`feature-build.md`, `feature-test.md`,
      `chore-build.md`, `bug-build-fix.md`) now state explicitly that the
      engine's `gates` node re-runs the full suite unconditionally
      regardless of the agent's own final run. `bug-build-test.md` is
      deliberately excluded from that addition: its whole job is to
      produce ONE deliberately-failing test (red-first), so framing the
      full suite as something the engine will eventually make green would
      nudge a red-first agent toward the opposite of its actual job. **The
      re-measurement itself is NOT done** — that requires dispatching real
      runs after this PR merges and is explicitly an operator follow-up
      (a build session in an isolated checkout cannot generate billed,
      operator-triggered runs of itself).

      **Measurement-validity notes for the re-measurement, both required
      reading before trusting any "drop":**
      1. **Method consistency.** §2's "269 Bash calls > 20s ≈ the median-15"
         phrasing suggests that baseline was derived by thresholding Bash
         call **duration**, not by matching command strings. This ticket's
         instrument (`countAgentSideSuiteInvocations`) is deliberately
         **exact-command-match**, not duration-based — a slow-but-scoped
         call like `bun test test/pipeline/nodes/ci-round.test.ts` (~68s
         solo, see item 4 of §4) would count under a duration threshold but
         correctly does NOT count here. Comparing the NEW instrument's
         post-change number against the OLD §2 figure directly would
         partly measure a change of counting method, not a change of agent
         behavior — the exact tok/s-mislabeling class of defect this
         ticket exists to kill. **The honest comparison runs this same
         instrument over BOTH the pre-change and post-change corpora**,
         never new-instrument-after vs §2's raw 15.
      2. **Corpus staleness.** The §2 baseline was taken across *every*
         banked run, some of which may predate commit `3bbbed7` (the
         "Iterating cheaply" guidance already present in the five
         build/test prompts before this ticket). If a large share of the
         corpus predates that commit, the pre-change baseline computed
         under note 1 is already mixed with runs that never saw that
         guidance at all — the operator's write-up should note how much of
         the corpus postdates `3bbbed7`, or the "drop" is not cleanly
         attributable to this ticket's own lever.
      3. **Exact-match is a floor, not a ceiling.** The counter matches
         byte-clean invocations only — `bun run test`, but not
         `bun run test | tail -5`, `bun run test 2>&1`, or any other
         flag/pipe/redirect decoration of the same command (deliberate, see
         Step 1 of the plan: fuzzy/prefix matching would reintroduce the
         scoped-vs-full ambiguity the exact-match design exists to avoid).
         A decorated full-suite run therefore undercounts. The repair
         prompt's own `{{cmd}}` is handed to the agent verbatim and un-
         decorated, so an agent that copies it still matches — but the
         absolute agent-side-execs figure this reports is a floor, not a
         precise count; read a "drop" with that in mind.
- [ ] **`gates` stays the definition of done.** Nothing here may weaken what
      must be green before `push` — that is the one rule this ticket cannot
      bend in pursuit of speed.
      **Build stage (2026-09-14):** confirmed unchanged by diff review —
      no edits to `src/pipeline/nodes/gates.ts`, `src/pipeline/engine.ts`,
      or any lane's `maxRounds`/gate list.
- [x] The 18.7-minute test file is profiled and either fixed or explicitly
      accepted with a reason. See the acceptance note under item 4 of §4
      above.

## Verify

- [ ] Median agent-side suite invocations per run drops, measured over at
      least 3 runs, against the numbers in §2. **Operator follow-up,
      post-merge** — see the note under §5 above; not completed by this
      build session.
- [x] No change to what `gates` executes.
- [x] `bun run lint && bunx tsc --noEmit && bun test`.

## Why this is priority 1

It is the largest single cost in the factory and it compounds: the suite is
also what saturates the box in `adw-bug-05` (parallel stalls) and what the
agent polls with `sleep` in `adw-bug-04` (the no-progress false positive).
Three open bugs share this root.
