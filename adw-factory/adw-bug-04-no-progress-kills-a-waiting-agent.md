---
id: adw-bug-04-no-progress-kills-a-waiting-agent
type: bug
status: done
priority: 2
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-bug-04-no-progress-kills-a-waiting-agent-1789428352450","branch":"adw/adw-bug-04-no-progress-kills-a-waiting-agent","workspace":"/Users/silouane/adw-factory/runs/adw-bug-04-no-progress-kills-a-waiting-agent-1789428352450/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/48","provider":"claude","model":"sonnet"}]
---
# The no-progress detector kills an agent for waiting on the gate suite it was told to run

> Reproduced from `adw-sync-01-reconcile-without-dispatching-1789372886529`.
> The run was doing correct work and had reached final verification.

## 1. What happened

```
⛔ no-progress — tool "Bash" repeated 3 times with no distinct successful
   action in between (input: {"command":"sleep 1"})
■ run-end blocked
```

The three `sleep 1` calls were not a stuck loop. Read the tool trace that
precedes them, from the run's own cLens capture:

```
Bash :: git status --short
Bash :: bun run lint && bunx tsc --noEmit && bun test 2>&1 | tail -60
Bash :: sleep 1
Bash :: sleep 1
Bash :: sleep 1          ← detector trips, run dies
```

The agent ran **the ticket's own definition of done** — the full suite — then
polled for it to finish. The same shape appears earlier in the same session:

```
Bash :: just test-fast > /tmp/test-fast.log 2>&1; tail -80 /tmp/test-fast.log
Bash :: just test-fast
Bash :: bun test test/intake test/observability …
Bash :: sleep 5
Bash :: sleep 1
```

**`sleep` here is the agent waiting, which is the correct behaviour.** The
detector cannot tell waiting from spinning, so it classified patience as a
loop and threw away a run at the finish line.

## 2. Why it happens, and it is not really about `sleep`

This repo's full suite is **~256 seconds** (measured 2026-09-14: 1599 tests,
256.07s). That is long enough that a long-running Bash call gets backgrounded
or times out, and the only thing an agent can then do is poll. So this is
structural, not a one-off: **any ticket whose verify step runs the full suite
can reach this state.**

`adw-auto-02-no-progress-detector` is doing exactly what it was built to do.
The rule is too coarse, not wrong.

## 3. Requirements

- [ ] **A pure wait is not a repetition.** `sleep`, and any command whose only
      effect is to pass time, must not count toward the repeat budget. Name
      the set explicitly in code rather than pattern-matching loosely.
- [ ] **A poll that is making progress is not a loop either.** Re-reading a
      growing log file, or re-checking a job that changes state, is progress
      even though the command is identical. If the observed OUTPUT differs
      between repeats, that is a distinct action.
- [ ] **Do not simply raise the threshold.** Three identical no-op calls is a
      reasonable ceiling; the defect is the classification, not the number.
      Raising it hides the real loops the detector exists to catch.
- [ ] **An unbounded waiter must still die.** An agent that sleeps forever is a
      genuine stall — bound the total waiting time (the liveness watchdog's
      `stallMinutes` is the existing precedent, Art. VIII) rather than the
      repeat count.
- [ ] Journal which rule fired, so "why did it stop" stays answerable from
      artifacts alone (Art. VI).

## 4. Red tests

- [ ] A tool trace of three consecutive `sleep 1` calls → **does not** trip.
- [ ] Three identical `cat log.txt` calls whose outputs **differ** → does not
      trip.
- [ ] Three identical calls with identical output and no waiting → **still
      trips**, unchanged. This is the test that proves the detector was not
      simply defanged.
- [ ] A waiter that exceeds the total-wait bound → trips, with a reason naming
      the wait bound and not the repeat count.
- [ ] Red before any fix.

## Verify

- [ ] Replay `adw-sync-01-…-1789372886529`'s captured tool sequence through the
      detector: it must not trip. The capture is on disk, so this is a real
      fixture, not a hand-built one.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Note

`adw-sync-01` itself needs nothing — a later run landed it and PR #38 is
merged, ticket `status: done`. This ticket is about the detector, which will
trip the same way on the next ticket whose verify step runs the full suite.

**Related, and probably the cheaper fix:** `adw-gates-02-stop-paying-for-the-
suite-four-times` already exists and targets the same underlying cost. A suite
that does not take 256s does not provoke the polling in the first place.
