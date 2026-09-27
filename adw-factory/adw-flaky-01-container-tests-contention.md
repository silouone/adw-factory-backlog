---
id: adw-flaky-01-container-tests-contention
type: feat
status: done
priority: 2
created: 2026-09-12
depends: []
attempts: [{"runId":"adw-flaky-01-container-tests-contention-1789218456861","branch":"adw/adw-flaky-01-container-tests-contention","workspace":"/Users/silouane/adw-factory/runs/adw-flaky-01-container-tests-contention-1789218456861/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-flaky-01-container-tests-contention-1789223159434","branch":"adw/adw-flaky-01-container-tests-contention-2","workspace":"/Users/silouane/adw-factory/runs/adw-flaky-01-container-tests-contention-1789223159434/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-flaky-01-container-tests-contention-1789228426083","branch":"adw/adw-flaky-01-container-tests-contention-3","workspace":"/Users/silouane/adw-factory/runs/adw-flaky-01-container-tests-contention-1789228426083/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The container tests are flaky under their own concurrency, and they leak containers

> Minted 2026-09-12. The specific defect that triggered the `adw-fe-01` loop.
> `adw-auto-01` makes the factory survive it; this makes it stop happening.

## Evidence

- `provisionContainer — auth slots (adw-m4-03)` failed in one full-suite run
  and passed in another on the same tree, same machine, same day.
- The suite spins **12 concurrent `adw-t-ctr-*` containers**.
  **CORRECTED 2026-09-13:** not `bun test` file-level parallelism. Each of the
  three real-docker files drained its minted containers in a single `afterAll`,
  so a container minted by test #2 stayed alive until the file finished.
  `container.contract.test.ts` run ALONE climbs 0 → 15 live containers
  monotonically, then drops to 0 in one step — sampled with `docker ps` beside
  the runner. One sequential file gets there on its own.
- **14 orphaned `adw-t-ctr-*` containers were on the host**, three of them
  `Exited (255) 6 weeks ago`. They are not reclaimed.
- The failing assertion is about env/token placement and takes >5 s of real
  container work — exactly the shape that loses a race under load.

## Requirements

- [ ] Establish the actual failure mode first: run the container suite alone
      versus under parallel load and record the difference. **Do not "fix" it
      before it is reproduced** — a flake fixed by guesswork is a flake moved.
- [ ] Container-provisioning tests do not run concurrently with each other.
      Serialise them, or gate them opt-in the way the codex smoke tests already
      are (`ADW_CODEX_SMOKE=1`) — the precedent exists in this suite.
- [ ] Every test container is removed on **every** exit path, including a
      failing assertion and a thrown error. The 6-week-old survivors prove the
      current teardown is best-effort.
- [ ] `adw clean` reclaims stray `adw-t-ctr-*` containers, so a killed suite
      cannot leave the host dirty.

## Verify

- [ ] The container suite passes ten consecutive times alone.
- [ ] It passes under deliberate parallel load, or is demonstrably excluded
      from parallel execution.
- [ ] After a run in which a container test **fails**, no container survives.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Whether these tests should exist. They caught real defects (`adw-m4-03` auth
slots is a security boundary) — the problem is how they run, not that they run.

## Run log

**2026-09-12 — run `…-1789218456861` BLOCKED at ~9 min into `build`: liveness
watchdog, "no SDK stream message for 10 minutes".**

**Root cause was CONTENTION, not this ticket's code.** It ran fully concurrent
with `adw-auto-01-baseline-gate-snapshot` (started 22 seconds apart, both
container-heavy) for 66 minutes. The agent went silent on
`docker ps -a --filter "name=adw-" -q | wc -l` — a millisecond command that
never returned. In the sibling run, a single test file took **1124 s (18.7
min)**. Docker was starved.

**NOT salvaged — the tree does not compile.** `bun run lint` fails and
`bunx tsc --noEmit` reports
`test/workspace/container.contract.test.ts(761,39): error TS2769` — a
`"not-a-real-kind"` literal not assignable to `WorkspaceKind`. The agent was
mid-edit when killed. Reset to `queued`.

**Run this one ALONE.** It is the ticket about container contention; running it
beside another container-heavy run is self-defeating.

**2026-09-12 — run `…-1789228426083` BLOCKED at 108.3 min: turn ceiling,
662/600, refusing the `repair` agent node.** `caps-01`'s fix worked exactly as
designed — the deterministic tail continued through `gates` instead of dying at
the node boundary, and only the *agent* node was refused.

Node breakdown: plan 12.6m/43t · **build 63.0m/465t** · test 28.0m/154t ·
gates 4.8m (lint pass, typecheck pass, **test FAIL**).

**The diagnosis inverts the assumption that tests are the cost:**

| | |
|---|---|
| tool time | **18.4 min (17%)** |
| model time | **89.9 min (83%)** |
| test invocations | 43, **16.4 min total**, median **5 s** |
| full-suite runs | **ZERO** |
| tool calls | 613 (470 Bash, avg 2.3 s) |

The `test:unit` guidance (committed 17:49:33, run started 17:53:46, so it WAS
in effect) worked — the agent ran no full suite at all. **It did not help,
because tests were never the bottleneck.** 662 turns at 9.8 s/turn is.

**Not salvaged.** 7 files, +363/−24, incl. a new `container-test-support.ts`
and its test — but `gates` reports `test=fail`, so it is not green. Worktree
preserved at `runs/adw-flaky-01-container-tests-contention-1789228426083/workspace`.

**Prime hypothesis for the turn count, untested:** `targets/adw-factory.json`
sets no `systemPrompt`, so the default applies and agents still run with an
**empty** system prompt — without Claude Code's ~7 836 tokens of tool-use and
conciseness guidance. `adw-sysprompt-01` merged, so switching to the preset is
now a one-line config change. See the handoff.

---

## Resolution — 2026-09-13

Salvaged by hand from the third blocked run's preserved worktree
(`runs/adw-flaky-01-container-tests-contention-1789228426083/workspace`,
gates: lint=pass typecheck=pass test=fail) and landed on `main` as
`adw-flaky-01: bound a test container's life to its own test, not the file's`.

The agent's work was **correct**. It found this exact mechanism, by this exact
method, and recorded the same number (~15) in its own doc comment. The run was
then discarded on a turn ceiling (662/600) four nodes from the finish.

Measured on integrated `main`:

| | before | after |
|---|---|---|
| peak concurrent containers | 13–15 | **1** |
| full suite | 451 s, nondeterministic (0/1/2/3/6 failures on identical trees) | **297 s, 1040 pass / 0 fail** |

The one failure seen mid-salvage (`cli.container.test.ts` "blocked path") was a
pre-existing regression from PR #11, not from this work — fixed separately
(`push: S2.7 is unconditional again`).

Left `in-review` rather than `done`: the commits are on local `main` and have
not been pushed or reviewed.
