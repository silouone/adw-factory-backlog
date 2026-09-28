---
id: adw-time
type: epic
status: queued
priority: 1
created: 2026-09-28
depends: []
children: []
attempts: []
---
# EPIC — A run never fails on time it didn't spend: the deadline bounds the agent, not the machine

**Operator rule (2026-09-28): work must not fail just because tests were
slow.** Needs a spec. It amends the Art. V caps semantics (`caps.minutes`
today is one whole-run wall clock, `engine.ts:485`) and adds host-level
admission control.

## What happened (2026-09-28, measured from journals)

Eight lanes were dispatched within about 10 minutes on one host:

- 5 × adw-factory
- 3 × cmc (docker `npm test`)

Load average was about 11–23. The adw-factory suite (~4,100 tests) took
about 12–17 minutes per run.

| run | cap | agent nodes | factory-owned (baseline + gates) | died in |
|---|---|---|---|---|
| adw-store-02 | 90 m | ~57 m (plan 4.9, build 19.6, test 31.6, repair 0.6) | **~40 m** (baseline 11.2, gates 15.6 + 13.4) | 2nd gates, deadline |
| adw-bug-30 | 90 m | ~53 m (plan 25, fix 22.3, reviews 1.6, review-fix 3.2) | **~37 m** (baseline 19.4, gates 16.8) | review-fix, **after gates were green** |

- The baseline minutes are mostly **lock wait**. The sha-keyed baseline
  cache (`nodes/baseline.ts`) makes peers on the same base wait for the one
  computing it, and that wait counts against every waiter's deadline.
- Baseline also runs the suite `BASE_RUN_COUNT = 2` times for flake
  detection.
- bug-30 had green gates and was killed in a review-fix round. Finished,
  verified work was thrown away for the clock.

## The principle

The wall-clock cap exists to stop a **runaway agent** and its spend.
Factory-owned work is different:

- baseline and gate execution
- lock and queue waits
- push and publish

It is not agent runaway, and it already has its own bound
(`ExecOptions.timeoutMs`, 30-min default per exec). Charging it to the agent
deadline makes the outcome a function of host load, not of the work.

## Direction (to be specced)

1. **Split the budget.**
   - `caps.agentMinutes`: wall time while an agent node is active. This is
     the runaway guard; turns and tokens caps stay as they are.
   - A separate, generous `caps.hardMinutes` backstop for the whole run
     (hours, not minutes), so a truly wedged lane still ends.
   - Waiting (locks, slots, queue) counts toward neither.
2. **Never abort factory-owned work for the deadline.** A gate run in flight
   finishes (bounded by its own exec timeout).
   - If the agent budget is exhausted **after gates went green**, the lane
     still commits, pushes and publishes, with the PR body naming what was
     cut (for example: "review round 2 not run — agent budget").
   - Green work is never discarded.
3. **Host admission control, event-driven.**
   - A host-wide slot pool for heavy executions (test suites, docker). Size
     it from CPU count; lanes queue for a slot instead of thrashing.
   - The same pool gates dispatch: new lanes wait when the host is saturated.
   - Slot wait is journaled (`slot-wait {ms}`) so the board shows *queued*,
     not *slow*.
4. **Make gates cheaper to wait for.** Stream results so a red is seen early,
   run the affected tests first, and reconsider `BASE_RUN_COUNT = 2` when
   the cache is warm. Optimisation, not correctness; after 1–3.

## Cheap fix, now (operator-approved direction)

Raise `caps.minutes` on queued/requeued adw-factory tickets, e.g. 90 → 180
and 120 → 240. Caps are per-ticket frontmatter; targets carry no defaults.
This only buys headroom; it doesn't remove the load dependence.

## Done when (epic)

- Re-running the 2026-09-28 wave shape (8 concurrent lanes) ends zero runs
  blocked on time for work whose agents finished inside `agentMinutes`.
- A lane whose gates are green is never blocked by the clock.
- The journal can show, per run, agent time vs factory time vs wait time.
