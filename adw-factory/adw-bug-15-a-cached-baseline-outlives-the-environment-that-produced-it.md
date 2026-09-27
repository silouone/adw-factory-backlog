---
id: adw-bug-15-a-cached-baseline-outlives-the-environment-that-produced-it
type: bug
status: done
priority: 2
created: 2026-09-19
review: false
caps: {minutes: 90, turns: 450}
depends: []
attempts: []
---
# The baseline said the suite was green, from a cache entry recorded in a different environment

> Observed 2026-09-19 on `adw-render-01-foundation-1789771519506`, alongside
> `adw-bug-14`.

## What happened

The run's `baseline` node reported:

```json
{"sha":"c787c51…","ok":true,
 "gates":[{"name":"test","outcome":"pass"}],
 "runs":2,"flaky":[],
 "cacheHit":true}
```

`cacheHit: true` — the suite was **not re-run**. The entry was read from
`runs/baselines/<sha>.json`, recorded earlier by a different invocation.

`adw-auto-01` exists so an agent never chases a failure it did not cause: the
base's known failures are subtracted from the gate result. With the baseline
asserting `test: pass`, there was nothing to subtract — so 4 environmentally-
determined failures were attributed to the agent's diff, and `repair` was sent
after them three times.

## The defect

**A baseline is keyed by commit sha alone.** But whether this suite passes also
depends on the environment it ran in — `E2B_API_KEY` and `CLAUDE_CODE_OAUTH_TOKEN`
decide whether whole suites execute or skip (`test/workspace/e2b-gate.ts`,
`test/workspace/docker-gate.ts`), and Docker's availability decides the same for
the container suite.

Same sha + key present → the E2B suite runs and can fail.
Same sha + key absent → it skips and the suite is green.

One cache entry cannot honestly represent both. `adw-auto-03` made the baseline
deterministic **within** an environment (2 runs, flaky list); it did not make
the cache key aware that the environment is part of the answer.

## Requirements

- [ ] **R1 — the cache key includes the gating environment.** Which
      capability-gating variables were present (**names and presence only,
      never values**) plus the availability of the gates that probe for Docker.
      A run whose gating environment differs from the cached entry's is a cache
      **miss**, and the baseline is re-measured.
- [ ] **R2 — a pure key function.** Sha + a normalised capability fingerprint →
      cache key. Pure, testable without a filesystem.
- [ ] **R3 — old entries miss, they do not crash.** Existing
      `runs/baselines/*.json` have no fingerprint. Treat them as a miss and
      re-measure. **Never** treat a missing fingerprint as a match.
- [ ] **R4 — journal the reason.** The `baseline` event says whether it hit,
      missed, or missed *because the capability fingerprint changed* — that
      third case is the one this ticket exists for.
- [ ] **R5 — no values.** Only presence. A test must pin it.

## Order of work — Article I

Write the pure key-function test first (R2/R3), confirm red, then implement.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] Red output pasted into this ticket (Art. I).
- [ ] A test proves the same sha with a differing capability fingerprint is a
      **miss**.
- [ ] A test proves a legacy entry with no fingerprint is a miss, not a crash
      and not a false hit.
- [ ] A test proves no environment **value** is written to the cache file.

## Out of scope

- Re-running every existing cached baseline.
- Changing how many times `baseline` runs the suite (`adw-auto-03` owns that).
- The repair-prompt half of this failure — that is `adw-bug-14`.
