---
id: adw-perf-01-the-feat-lane-pays-for-the-same-context-three-times
type: feat
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-perf-01-the-feat-lane-pays-for-the-same-context-three-times-1789465948398","branch":"adw/adw-perf-01-the-feat-lane-pays-for-the-same-context-three-times","workspace":"/Users/silouane/adw-factory/runs/adw-perf-01-the-feat-lane-pays-for-the-same-context-three-times-1789465948398/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/52","provider":"claude","model":"sonnet"}]
---
# A 41-minute median run against a 5–10 minute hand-built equivalent — where the other 30 minutes go

> Operator, 2026-09-15: *"If I were to build that here directly in a Claude
> session most builds would take 5 to 10 min max, avg is now 45min+, something
> is not right… we must find the culprit."*
>
> Measured across **42 green runs over 5 minutes**. There are three culprits,
> none of them the model.

## 0. It is NOT tok/s — rule this out first

Output tokens divided by **model** time, not wall time:

| node | tok/s (wall) | tok/s (model only) |
|---|---|---|
| plan | 53.3 | 53.9 |
| build | 32.7 | **53.3** |
| test | 18.1 | **42.0** |
| build-fix | 21.3 | **54.5** |

**Throughput is a healthy flat ~40–55 tok/s on every node.** Anything that
looks slower is blocked, not slow. `metrics.ts`'s `tokensPerSecond` divides by
WALL time, which is why the number on screen looked alarming — that is
`adw-bug-06` §1.

## 1. Where 41 minutes actually goes

Median green run, by node:

| node | median | share of node time |
|---|---|---|
| **build** | **18.2m** | 42.6% |
| **test** | **12.9m** | 16.4% |
| **plan** | **9.4m** | 16.8% |
| gates | 3.9m | 8.9% |
| baseline | 3.9m | 5.4% |
| everything deterministic | <10s total | 0.3% |

The deterministic control plane costs essentially nothing. **Three sequential
agent stages cost 40 minutes.**

## 2. Culprit A — the feat lane is three cold sessions

A hand-built equivalent is **one** session that reads the code once and keeps
it. The feat lane is `plan → build → test`, each a **fresh SDK session with no
`resume`**, each re-deriving the same codebase from zero.

Measured **time to first edit** — how long a stage explores before it writes
anything:

| stage | session | time to first edit | tool calls first |
|---|---|---|---|
| `build` | cold | **177s** | 18 |
| `test` | cold | **295s** | 18 |
| `build-fix` | **RESUMES** `build-test-only` | **126s** | 8 |

**`build-fix` is 2.3× faster to first edit than `test`, and needs less than
half the tool calls, because it resumes a session that already holds the
context.** The bug lane does this deliberately — the README calls it out: *"a
cold restart would throw away the context in which the agent decided the
test's shape."* `grep -c resume src/pipeline/lanes/feat.ts` returns **0**.

So the feat lane pays the discovery cost three times, and ~8 minutes per run
is one agent re-reading what the previous agent just read.

- [ ] **`build` resumes `plan`'s session** where the provider supports it, or
      the plan artifact is made rich enough that build does not re-explore.
- [ ] **`test` resumes `build`'s session.** This is the biggest single win on
      the board: 12.9 minutes for a stage whose job is "expand coverage" on
      code the previous session just wrote.
- [ ] Reuse the existing seam — `build-fix` already resumes via
      `ctx.data.sessionId` (`bug.ts:303`). Do not build a second mechanism
      (Art. VIII).

## 3. Culprit B — 31% of every run is the test suite

| | median |
|---|---|
| suite inside deterministic nodes (`baseline`, `gates`, `red-check`) | 5.2m |
| suite run **by the agent** inside `build`/`test`/`repair` | 7.6m |
| **total** | **12.8m = 31% of the run** |

`adw-bug-06` owns this. Noted here because the two compound: a slow suite is
worse when three stages each run it.

## 4. Culprit C — is the `test` stage earning its keep?

`test` costs **12.9 minutes**, spends **5 of them re-reading code `build` just
wrote**, and the `bug` lane has no equivalent stage at all.

- [ ] **Answer it with evidence, not opinion.** For the last N green feat runs,
      measure what `test` actually added: tests written, coverage delta, bugs
      caught that `build` missed. The data is in the captures and the diffs.
- [ ] If it earns its keep, resume it into `build`'s session (§2) and it gets
      cheap. If it does not, fold it into `build`'s prompt and delete the stage.
- [ ] **This is an operator call, not a build decision** — `test` is a lane
      stage, and removing it changes what the feat lane means.

## 5. What good looks like

If §2 and §3 land: plan stays ~9m, build keeps its real work but loses ~3m of
rediscovery and ~4m of redundant suite runs, test either disappears or drops to
~6m. **A 41-minute median becomes roughly 20–25.** That is the honest target —
not 5–10 minutes, because the factory also runs a baseline, runs the gates,
produces a reviewable plan and opens a PR, none of which a hand session does.

The gap worth closing is the **waste**, not the rigor.

## Verify

- [ ] Median green-run wall clock, measured over at least 5 runs before and
      after, recorded in this ticket.
- [ ] Median time-to-first-edit for `test` drops toward `build-fix`'s 126s.
- [ ] `gates` still runs the full suite — nothing here weakens the definition
      of done.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Related

`adw-bug-06` (the suite is 78% of tool time) · `adw-bug-05` (the ~300s
idle-stream timeout, which is what makes a long `plan` generation fatal).
Together these three are the whole throughput story.
