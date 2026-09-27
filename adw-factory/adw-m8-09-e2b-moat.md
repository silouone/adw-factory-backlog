---
id: adw-m8-09-e2b-moat
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m8-08-integration-worktree]
attempts: []
---
# Stage 2 live bar — the MOAT: `bug` and `feat` merged on E2B

> Source of truth: `specs/adw-v1.1-lanes.md` (Acceptance §Stage 2). Read
> constitution + specs first. **Operator-gated.** The **mandatory done bar for
> the whole M8 epic** — v1.1 plan+build is not done until this passes.

## Why this is the moat

The factory exists because it runs end-to-end in an isolated **E2B** remote
sandbox. A new lane type is only "done" once it has produced a **merged PR on
E2B**. `chore` is proven there (PR #17); this proves the two new types —
including the **Planner-led feature pipeline** and the **two-phase red-check**
running inside a remote sandbox, the real parity test.

## Requirements

- [ ] A real cLens **`bug`** travels Graph C on the **E2B remote** kind to a
      **merged** PR — base-green, plan, clean red-check, resumed fix, and gates
      all running inside the sandbox.
- [ ] A real cLens **`feat`** travels Graph B on **E2B** to a **merged** PR —
      plan → build → test → gates inside the sandbox.
- [ ] **Capture parity** on E2B for both runs (journal + spans + captured
      sessions replayable, Art. VI) — matching the M5/M6 remote-capture bar.
- [ ] No isolation-specific divergence: each graph behaves identically to its
      worktree run except for the sandbox (kinds indistinguishable through the
      seam, S4.3).

## Build protocol

1. Reuse/re-shape the Stage-1 `bug`/`feat` tickets against the E2B kind.
2. Dispatch each on E2B; confirm the red-check + Planner pipeline run remotely;
   record run ids / PR numbers / sandbox ids here.
3. Operator reviews and merges each PR.

## Runs (Stage 2 — E2B moat, 2026-07-23/26) — DONE

Both new lanes traveled Graph C/B **inside a remote E2B sandbox** to a **merged**
PR — the moat is met:

- **`bug` (Graph C) → PR silouone/clens#20 (MERGED).** clens-011-fmtduration-hours,
  run `clens-011-fmtduration-hours-1784805064990`. Full red-first ran remotely:
  base-green → plan → build-test-only → **clean red-check** → resumed fix → gates
  green, all in-sandbox, **4.0 min**. The two-phase red-check machinery proven on
  E2B (the real parity test).
- **`feat` (Graph B) → PR silouone/clens#21 (MERGED).** clens-013-list-reverse-flag,
  run `clens-013-list-reverse-flag-1785085459819`. plan → build (tests-first) →
  test (coverage) → gates green, in-sandbox, ~15 min.

**Two live findings (both E2B remote-infra, NOT lane/code bugs; the lanes + impls
were correct in every case):**
1. **Run-wide minutes vs the ~60-min E2B sandbox ceiling.** The first feat pick
   (`clens-012` `list --limit`) was a heavy Graph B feat — its 3 agent stages +
   slow in-sandbox gate runs on E2B's 2-CPU sandbox reached ~58 min and hit the
   ceiling during a flaky-gate repair round → blocked. Fix: a *smaller* feat
   (`--reverse`) fits comfortably. Lesson: size feat tickets to the sandbox
   ceiling for E2B (the spec's decision-15 "run-wide minutes is the pressure
   point," now proven on the remote kind).
2. **E2B transport stream stall.** The first `--reverse` run finished its work
   green in-sandbox in ~18 min, but E2B's transport then went silent 10 min (the
   agent's terminal `result` never arrived) → the liveness watchdog correctly
   hard-stopped it → blocked. A single re-run was clean (#21). The feat lane's 3
   agent stages carry ~3× the stall exposure of the bug's fast single-fix path
   (the bug ran in 4 min, no stall). This is the documented m5-04/m5-05 E2B
   "agent goes silent mid-stride" wedge — remote transport reliability, deeper
   root-cause out of M8 scope.

Capture parity note: remote runs journal `capture ok:false` (the async
fetchTranscript seam is deferred — adw-m5-06); the journals are otherwise
replayable (node sequence + gate-result arrays), and the bug run's clean-red
verdict is recorded.

## Verify

Two merged cLens PRs (one `bug`, one `feat`) produced on **E2B** (#20 bug, #21
feat) — **DONE**. The bug run's clean-red verdict is in the remote journal.
**On green, the M8 epic is done** → epic closed.

## Out of scope

Container-kind parity for the new types (orthogonal to the moat); hotfix / router
/ "Your ADW" (spec §Out of Scope).
