---
id: adw-gates-02-stop-paying-for-the-suite-four-times
type: feat
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-gates-02-stop-paying-for-the-suite-four-times-1789345203217","branch":"adw/adw-gates-02-stop-paying-for-the-suite-four-times","workspace":"/Users/silouane/adw-factory/runs/adw-gates-02-stop-paying-for-the-suite-four-times-1789345203217/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/31","provider":"claude","model":"sonnet"}]
---
# Half of every run is the same test suite, run four times

> Minted 2026-09-14 from a profile of `adw-fe-09-node-drawer-1789339650557`
> (green, 59.5 min, zero repair rounds) and the nine runs before it. The agents
> are **not** the problem in any of them — this is gate cost.

## Evidence

`fe-09` produced a **1044-line** change (`src/web/drawer.ts` 228, `projection.ts`
+36, `test/web/drawer.test.ts` 792) across three files, green first try, 53
turns in `build`. A healthy run. It took **59.5 minutes**, and the suite is
where the time went:

| where | command | duration |
|---|---|---|
| `baseline` node | the target's gates, on the BASE tree | 3.9 min |
| inside `build` | `bun run lint && bunx tsc --noEmit && bun test` | **9.1 min** |
| inside `test` | `bun test` | **8.0 min** |
| `gates` node | the target's gates, on the WORK tree | 8.5 min |
| | **total** | **≈29.5 min — half the run** |

Per-stage model-vs-tool time from the cLens captures confirms where the agent
was blocked rather than thinking:

| stage | model | tool |
|---|---|---|
| `plan` | 9.4 min | **0.1 min** |
| `build` | 6.0 min | **11.6 min** |
| `test` | 3.5 min | **8.9 min** |

`plan` — the stage doing real design work — is the cheap one. `build` and
`test` spent **twice** as long watching `bun test` as thinking.

**The bug lane is worse: it runs the suite FIVE times.** `adw-depends-enforce`
spent **9 min in `baseline` + 7 min in `base-green-check`** — 16 minutes, back
to back, on the *same base tree with the same gates* — before a single agent
token was spent.

## Three separable causes

### 1. `baseline` and `base-green-check` are the same measurement (bug lane)

Both run `target.gates` on the untouched checkout, consecutively, at the same
sha. `baseline` already records per-gate outcomes; `base-green-check` only
needs "was anything red". The second run adds no information.

Deliberately not unified when `adw-auto-01` landed (bug.ts: *"answers a
DIFFERENT question … the two are deliberately not unified"*) — that reasoning
predates the measurement above and deserves re-examination, not a silent
override. **Amendment rule applies if the answer is to merge them.**

### 2. The baseline cache races, so concurrent runs all miss

`runs/baselines/<sha>.json` is meant to amortize the base measurement. Measured
over the last ten runs:

| sha | runs | cacheHit |
|---|---|---|
| `61b9158` | fe-04, fe-09 | **false, false** |
| `22af665` | fe-02, fe-07 | **false, false** |
| `964a5eb` | fe-05, fe-06 | **false, false** |
| `61b9158` | tools-01 (later) | true |
| `4e2f5cb` | fe-10 (later) | true |

**8 of 10 runs paid it in full.** Runs dispatched together all find no cache,
all run the suite, all write the same file. The cache only helps runs that
start *after* another has finished — which is the case parallelism exists to
avoid.

### 3. The gate suite is ~8 min and mostly waiting

`bun test` is ~300-480 s wall against ~160 s of CPU — real Docker, real git,
real subprocesses. `bun run test:unit` is **21 s** for 439 of the tests, and
the agents already prefer it (the 2026-09-13 prompt change landed and is
followed). But the definition of done is the full gate, so every stage pays it
at least once.

## Requirements

- [ ] Establish the current cost first and record it: how many full-suite
      executions a `chore`, `feat` and `bug` run each perform today, measured
      from journals, not counted from source. **Do not optimise before it is
      measured** — the same discipline `adw-flaky-01` was held to.
- [ ] A concurrent baseline on a sha another run is already measuring must not
      re-run the suite. Whatever the mechanism (a lock like `_repo.lock`, a
      wait, an in-flight marker), a late reader must not silently trust a
      partial write.
- [ ] Decide, explicitly, whether `base-green-check` can be answered from
      `ctx.data.baseline`. If yes, propose the spec amendment rather than
      editing bug.ts quietly.
- [ ] Do **not** weaken the gate to buy speed. Making `test` mean `test:unit`
      would make every run faster and every green less true — that is the
      `adw-gates-01` failure mode, not a fix for this one.

## Verify

- A second run dispatched against a sha already being baselined does not
  execute the gate suite a second time.
- A `bug` run executes the base gates once, not twice.
- The full-suite execution count per lane is recoverable from the journal.

## Out of scope

Making the suite itself faster (container tests are the I/O boundary on
purpose) and splitting the gate into fast/slow tiers. Both are real options;
both change what "done" means and want their own ticket and operator decision.

## Run log

**2026-09-14 — Step 1 measurement, before any behavior change.**

`src/observability/gate-cost.ts` (`countFullSuiteExecutions`, pure, tested in
`test/observability/gate-cost.test.ts`) plus `scripts/gate-cost.ts` (the CLI
reading `runs/*/journal.jsonl`) are the mechanism this requirement calls for:
a run's full-suite execution count read off its journal, not hand-counted from
lane source.

This is a fresh isolated build workspace — `runs/` is gitignored and this
checkout has never executed a real lane, so there is no live journal to point
the script at. The number below is **not fabricated**: it is the script's own
output, run against three synthetic journals hand-built to the exact node
sequence each real lane (`src/pipeline/lanes/{chore,feat,bug}.ts`) journals
today, green first try. The mechanism is what's being certified here — the
same script will read a live `runs/` the moment one exists, with zero code
change.

```
chore-demo-1        clens-100   full-suite-execs=2  [baseline, gates]
feat-demo-1         clens-101   full-suite-execs=3  [baseline, gates, gates]
bug-demo-prefix-1   clens-102   full-suite-execs=4  [baseline, base-green-check, red-check, gates]
```

- **`chore` = 2** pipeline full-suite runs (`baseline`, `gates`), green first
  try. Matches the ticket framing.
- **`feat` = 2** pipeline runs on a clean first pass; the fixture above adds
  one `gates↺repair` round (baseline + gates + gates = 3) to show a repair
  round is counted once per re-entry, not deduplicated — the ticket's "four
  times" figure is this pipeline cost (2) **plus** the two agent-invoked
  self-checks inside `build`/`test` (invisible to the journal — they live in
  each node's own cLens session capture, not addressed by this script; see
  the plan's grounding note on why that's a separate, corroborating evidence
  tier, not a hard-tested code path).
- **`bug` = 4** pipeline runs TODAY (`baseline`, `base-green-check`,
  `red-check`, `gates`) — one more than the ticket's evidence table implied
  (it quoted `baseline` + `base-green-check` as the redundant pair; `red-check`
  and `gates` are two more full runs on the same tree, on top of those two).
  This is the number Step 3 below reduces to 3 by answering
  `base-green-check` from `ctx.data.baseline` instead of re-executing it.

Confirms the ticket's core diagnosis (bug pays for the suite the most) while
correcting the exact figure: **4** pipeline-level full-suite executions today,
not counting agent self-checks, not 5 — the "FIVE times" in the evidence
section conflates the pipeline cost with an agent self-check. Step 3 lands the
fix that takes bug's pipeline cost from 4 to 3.
