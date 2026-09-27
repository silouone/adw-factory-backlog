---
id: adw-gates-01-silent-skip-shrinks-done
type: bug
status: done
priority: 1
created: 2026-09-13
depends: []
attempts: [{"runId":"adw-gates-01-silent-skip-shrinks-done-1789345217895","branch":"adw/adw-gates-01-silent-skip-shrinks-done","workspace":"/Users/silouane/adw-factory/runs/adw-gates-01-silent-skip-shrinks-done-1789345217895/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/28","provider":"claude","model":"sonnet"}]
---
# A gate that silently skips half its tests still exits 0, so "done" shrinks without anyone noticing

> Minted 2026-09-13 from the first run under the new config
> (`adw-obs-02-cache-token-mirror-1789309935236` — green, PR #12).

## What happened

Docker was down for the whole run. `dockerGate()` does exactly what it was
built to do — skips loudly to **stderr**:

```
⚠ SKIPPING Workspace contract: container: docker is not available —
  the container-kind tests did NOT run (adw-m4-02 verify requires them for real)
```

…and then `bun test` **exits 0**. The gates node reads only the exit code, so:

- `baseline` recorded `test: pass`,
- `baseline-green-check` advanced,
- `gates` recorded `test: pass`,
- `push` and `open-pr` ran, and the run ended **green**.

~47 tests never executed, in either the baseline or the gate, and nothing in
the journal says so.

## The evidence it is a timing signature, not a guess

Node durations from the run's own journal, against measurements on the same
machine the same day:

| | duration |
|---|---|
| `baseline` (gate suite on the base) | **150.0 s** |
| `gates` (gate suite on the work) | **143.2 s** |
| measured: full suite, docker DOWN | **133.7 s** (997 tests, 5 loud skips) |
| measured: full suite, docker UP | **297 s / 329 s / 484 s** (1044 tests) |

## Why it is worse than one green run

The baseline is **cached by base sha** at `runs/baselines/<sha>.json`. This run
wrote `{"sha":"1ad7773…","ok":true,"gates":[…,{"name":"test","outcome":"pass"}]}`
— a docker-less measurement — and **every future run on that sha would have
reused it, including runs with docker back up.** (Deleted by hand 2026-09-13;
the mechanism is not.)

This is the `adw-auto-03` poisoning hazard arriving by a second route: not a
flaky test, an absent dependency.

## Requirements

- [ ] Establish the failure mode first: a gate whose command exits 0 while a
      documented capability was unavailable must be **detectable from the
      journal alone**. Decide what evidence the factory has to work with —
      exit code only, or stdout/stderr, or a target-declared precondition.
- [ ] A gate that skipped part of itself must not silently satisfy the
      definition of done. Refusing, warning-in-the-journal and marking the PR
      body are all defensible; silently passing is not.
- [ ] A baseline measured under a degraded environment must not be cached as
      if it were complete — or must record what was unavailable, so a later
      run with the capability back does not reuse it.
- [ ] `just doctor` already reports `docker: down`. The factory never consults
      it. Decide whether dispatch checks preconditions (and which), or whether
      this stays purely a gate-level concern.

## Verify

- With docker stopped, a run on a target whose `test` gate needs it does NOT
  end green with no trace.
- A baseline written with docker down is not reused by a run with docker up.

## Out of scope

Making the container tests not need real docker. They are the I/O boundary on
purpose (same law as the worktree suite's real git); this ticket is about the
factory noticing, not about removing the dependency.
