---
id: adw-gates-06-an-unparsed-runner-cannot-trip-the-livelock-check-2c9572
type: bug
status: queued
priority: 2
created: 2026-09-27
review: false
caps: {minutes: 90, turns: 400, stallMinutes: 25}
depends: [adw-gates-05-jest-failures-are-named-181b39]
attempts: []
---
# An unparsed test runner must not be able to trip the repair loop's livelock check

> Split out of adw-gates-05 R3 on 2026-09-27: the factory run implemented R1/R2
> (jest names parsed) and, correctly under Art. I, declined R3 because the
> test-only stage had written no red test for it. R3 needs engine changes and its
> own red tests; it gets its own ticket.

## The defect

`gateFingerprint` (`src/pipeline/engine.ts`, adw-cost-00 R2) is gate names ×
outcomes plus `failingTests`. When a runner's output is not parsed (`[]`), two
different failures fingerprint identically and the loop stops after two rounds —
measured on `cqc-fe-01-…-1790524928011`: round 2 had 9 failing jest tests, round
3 had 6 with different names, and the run was blocked as "not converging". Jest
is parsed since adw-gates-05; the next runner (vitest, mocha, go test…) will hit
the same wall.

## Requirements

- [ ] **R1** When the test gate fails with `failingTests: []` and non-empty
      stdout, `gateFingerprint` includes a stable hash of the gate's failure tail
      (the last N lines of stdout), so unparsed-but-different failures do not
      collide. Parsed lists keep today's behaviour byte-identically.
- [ ] **R2** The journal's `round`/livelock event records which fingerprint branch
      was used (`names` | `tail-hash`).

## Verify

- [ ] Red test: two `GateResult`s with `failingTests: []` and different stdout
      tails → different fingerprints; identical tails → identical. RED today.
- [ ] Red test: the engine, fed the cqc-fe-01 round-2 and round-3 outputs with
      an empty parsed list, does NOT stop the loop between them.
- [ ] Existing engine/gates suites green unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
