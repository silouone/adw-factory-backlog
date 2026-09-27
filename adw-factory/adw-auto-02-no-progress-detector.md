---
id: adw-auto-02-no-progress-detector
type: feat
status: done
priority: 1
created: 2026-09-12
depends: []
attempts: [{"runId":"adw-auto-02-no-progress-detector-1789313146156","branch":"adw/adw-auto-02-no-progress-detector","workspace":"/Users/silouane/adw-factory/runs/adw-auto-02-no-progress-detector-1789313146156/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/14","provider":"claude","model":"sonnet"}]
---
# Detect an agent looping with no progress, and stop it — the generic backstop

> Minted 2026-09-12 alongside `adw-auto-01`. That ticket removes the known
> cause of the `adw-fe-01` loop; this one catches the loops nobody predicted.

## Why a second mechanism

The factory bounds agents three ways — repair rounds, a turn ceiling, and a
wall clock — and **all three are budgets, not progress checks.** A loop that
stays inside every budget is invisible until the budget runs out, and then the
run dies with nothing to show. `adw-fe-01` burned 47 minutes that way.

The liveness watchdog does not help: it detects **silence**. A looping agent is
noisy — it is working hard, repeatedly, on the same thing.

## The signal

The factory already forwards **every** hook event to the cLens sink, so the
tool stream is available at a seam that exists today. A loop is visible in it
without any semantic judgement:

> the same tool with the same input, returning the same result, N times, with
> no distinct successful action in between.

That is a pure function over a rolling window. In the `adw-fe-01` case it would
have fired on the third identical `bun test` — at ~15 minutes instead of 47.

## Requirements

- [ ] A **pure** detector: a rolling window of `(tool, normalised input,
      result hash)` in, a `no-progress` verdict out. No I/O, no clock (Art. IX)
      — the window and threshold are parameters.
- [ ] Normalisation is deliberate and documented: whitespace and volatile
      fields (timestamps, durations, run ids, temp paths) are excluded from the
      hash, or a loop that differs only by elapsed time will never match.
- [ ] Threshold is a named constant with its rationale. **3 identical
      repetitions** is the starting point — two can be legitimate (run, fix,
      re-run).
- [ ] On a trip: journal a `no-progress` event naming the repeated command and
      the repetition count, then **abort the node** the way the liveness
      watchdog does. The run ends `blocked` with a reason an operator can act
      on, not a budget exhaustion message that says nothing.
- [ ] Wired at the existing hook-forward seam — **not** a new capture channel.
- [ ] Tolerant: a detector fault is journaled and never aborts a healthy run.

## Verify

- [ ] Three identical (tool, input, result) triples in the window → trips.
- [ ] The same command with a **different** result (a test suite going from 1
      fail to 0) → does **not** trip. This is the false-positive that matters:
      an agent legitimately re-running a suite it just fixed.
- [ ] Interleaved distinct work resets the window.
- [ ] Volatile fields differing (timestamps, durations) still trips — proves
      the normalisation.
- [ ] A throwing detector leaves the run unaffected.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Judging *why* the agent is looping, or intervening to fix it. This stops the
burn and hands a legible reason to the operator; it does not repair.
