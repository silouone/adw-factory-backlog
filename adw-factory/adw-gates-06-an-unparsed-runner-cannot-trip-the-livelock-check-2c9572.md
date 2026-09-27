---
id: adw-gates-06-an-unparsed-runner-cannot-trip-the-livelock-check-2c9572
type: bug
status: in-progress
priority: 2
created: 2026-09-27
review: false
caps: {minutes: 90, turns: 400, stallMinutes: 25}
depends: [adw-gates-05-jest-failures-are-named-181b39]
attempts: [{"runId":"adw-gates-06-an-unparsed-runner-cannot-trip-the-livelock-check-2c9572-1790538681667","branch":"adw/adw-gates-06-an-unparsed-runner-cannot-trip-the-livelock-check-2c9572","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-06-an-unparsed-runner-cannot-trip-the-livelock-check-2c9572-1790538681667/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/136","provider":"claude","model":"sonnet"}]
---
# An unparsed test runner must not be able to trip the repair loop's livelock check

> Split out of adw-gates-05 R3 on 2026-09-27: that run implemented R1/R2 (jest
> names parsed) and, correctly under Art. I, declined R3 because the test-only
> stage had written no red test for it. R3 needs engine changes and its own red
> tests, so it gets this ticket.
>
> **Amended 2026-09-27, same day, before any attempt.** The original R1 hashed
> "the last N lines of **stdout**" and only did so "when `failingTests: []`".
> Both halves are wrong — **R1a** and **R1b** below supersede them and say why.
> `review: false`: hard-gated.

## The defect

`gateFingerprint` (`src/pipeline/engine.ts:1140-1144`, adw-cost-00 R2) is gate
names × outcomes plus `failingTests`. When a runner's output is not parsed the
list is `[]`, so **two different failures fingerprint identically** and the
engine stops the loop after two rounds with *"gate outcome repeated across
rounds, not converging"*.

Measured on `cqc-fe-01-drawer-keeps-keyboard-focus` (target `cmc`, provider
codex, gates in Docker `node:14.21.3`) — blocked **twice**, both times on the
identical signature `lint:pass|test:fail[]`:

| run | round | what actually failed |
|---|---|---|
| `…-1790524928011` | 1 | `lint` — tslint `prefer-for-of` at `Drawer/index.tsx:302` |
| `…-1790524928011` | 2 | 9 failing jest tests, five distinct `●` names |
| `…-1790524928011` | 3 | **6** failing tests, a *different* set (re-run in the kept workspace) |
| `…-1790532937437` | 1 | suite never ran: `TS2345 keyCode does not exist in KeyboardEventInit` |
| `…-1790532937437` | 2 | same signature → blocked |

The first run was converging and was stopped one round early. In both runs the
repair prompt carried the full runner output, so the **agent** was not blind —
the **factory** was.

## What the livelock check actually needs

Not test names. A *discriminator*: "did this round produce a different result
than the last?" Names are one possible discriminator and the worst-behaved one,
because producing them requires the factory to know every runner's reporter
format. A digest of the failure output is framework-agnostic and strictly
better: it also distinguishes *same tests failing with a different error*, which
is real progress that a name list calls stuck.

Per-test identity is genuinely needed in exactly one other place —
`classifyGateOutcome`'s `introduced = current − baseline` — and only when the
baseline is **red**. `cqc-fe-01`'s baseline was `[lint pass, test pass, build
pass]`, so `baselineFailingTests = []` and every failure was `fail-introduced`
regardless of names. That is `adw-gates-07-…-ad75a0`'s problem, not this one.

## Requirements

- [ ] **R1** `gateFingerprint` includes a stable hash of the failing gate's
      **normalized failure output**, computed from the retry payload —
      `GateFailureReport` (`src/pipeline/nodes/gates.ts:52`), which already
      carries `stdoutTail` **and** `stderrTail` and is already in scope at the
      call site as `result.payload` (`engine.ts:960`). `payload` is typed
      `unknown`; add a type guard beside the existing `hasGates` /
      `hasEnvDivergence` / `isEnvDivergence` guards (`engine.ts:1150-1155`).
- [ ] **R1a** **Both streams, not stdout.** Measured 2026-09-27 against
      `…-1790532937437`'s real gate output: `parseFailingTests(stdout)` → `[]`,
      `parseFailingTests(stderr)` → `["src/components/Drawer/index.test.tsx"]`.
      jest and bun both write their failure lines to **stderr**; pytest is the
      only one of the three that uses stdout. A stdout-only tail hash would
      digest jest's per-file coverage table and never see the failure.
- [ ] **R1b** **Unconditional.** The digest always participates, whether or not
      `failingTests` is populated. This supersedes the original R1's "when
      `failingTests: []`" condition and its "parsed lists keep today's behaviour
      byte-identically" promise — the whole point is that the livelock check
      must never again depend on whether a runner happens to be parsed. The next
      runner (vitest, mocha, `go test`, `cargo nextest`) then needs **no factory
      change at all**. Keep the `name:outcome` prefix in the signature string
      for legibility; it simply stops being the sole discriminator.
- [ ] **R1c** **Normalize, so an unchanged failure still collides.** A rerun of
      the same failure must hash identically. Scrub at minimum: per-test
      durations (`[0.13ms]`, `(12 ms)`), summary timings (`Time: 1.2 s`), ISO
      timestamps, absolute workspace paths (the run id is in every path), and
      jest's per-file coverage percentage table. Each scrubber is generic text
      handling, **not** per-framework knowledge — if one needs to know which
      runner produced the output, it is the wrong scrubber.
- [ ] **R1d** **Fail safe toward waste, never toward a false block.** Under-
      normalized → the fingerprint differs every round → the check never fires →
      the run spends its full 3 rounds. That is wasted spend and is acceptable.
      Over-normalized → two genuinely different failures collide → a converging
      run is blocked, which is the bug being fixed. When in doubt, scrub less.
- [ ] **R2** The journal's `round` / livelock event records the short digest, so
      *"did the failure actually change between rounds?"* is answerable from the
      journal alone. Today it records only `failingTests: []`, and the terminal
      round's output is never captured anywhere at all — the `…-1790524928011`
      round-3 evidence in the table above had to be recovered by re-running the
      gate by hand in the kept workspace.

## Files this ticket owns — parallel-safety

`adw-gates-07-a-target-declares-how-its-runner-reports-failures-ad75a0` is
fired **concurrently** with this ticket. They are disjoint by construction:

- **This ticket touches only** `src/pipeline/engine.ts` and its tests.
- It must **not** edit `src/pipeline/nodes/gates.ts`, `src/observability/journal.ts`,
  the three failing-test parsers, or `src/targets/loader.ts` — gates-07 owns
  those.
- It must **not** add a field to `GateResult`. None is needed: the digest comes
  from the retry payload, which is already at the call site. Adding one would
  put this ticket inside `gates.ts` and `journal.ts` and guarantee a conflict.

If implementation seems to require touching a gates-07 file: **stop and propose
a spec amendment** (project CLAUDE.md) rather than reaching across the fence.

## Verify

- [ ] Red first (Art. I), all of these RED today:
  - two gate failures with identical `failingTests: []` and **different**
    stderr tails → **different** fingerprints;
  - two whose tails differ **only** in durations/timestamps/paths → **identical**
    fingerprints (R1c);
  - the engine fed `…-1790524928011`'s round-2 and round-3 outputs with an empty
    parsed list does **not** stop the loop between them;
  - a failure whose names are populated *and* whose output changed → different
    fingerprints (R1b — this is the case the original conditional R1 missed).
- [ ] The existing engine suite is green **unmodified**, including the
      six-field retry-payload pin (`gates.ts` decision 9).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

Reading machine-readable runner reports, the `Gate.report` field, and deleting
the stdout regex parsers — all `adw-gates-07-…-ad75a0`. Banking the terminal
round's gate output as an artifact (R2's digest answers the question this
ticket needs; the artifact gap is worth its own ticket).
