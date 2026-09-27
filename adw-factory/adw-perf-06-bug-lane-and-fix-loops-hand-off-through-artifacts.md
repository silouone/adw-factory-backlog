---
id: adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts
type: feat
status: done
priority: 2
created: 2026-09-27
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: [adw-perf-05-steps-are-isolated-and-the-plan-is-the-contract]
attempts: [{"runId":"adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790476027269","branch":"adw/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790476027269/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790480156025","branch":"adw/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790480156025/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790482383670","branch":"adw/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-3","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790482383670/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790486509641","branch":"adw/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-4","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790486509641/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The bug lane and the fix loops hand off through artifacts, not shared sessions

Second half of the step-isolation rule set by
`adw-perf-05-steps-are-isolated-and-the-plan-is-the-contract`: **each agent
step is a fresh session, and only files in `.adw/artifacts/` (plus the ticket
and the repo on disk) cross a step boundary.**

## Remaining cross-step resumes (verified 2026-09-27)

| site | today | after |
|---|---|---|
| bug lane `build-fix` (`bug.ts`, both call sites) | resumes `build-test-only`'s session | fresh session: ticket + `plan.md` + the reproducing test file(s) + the red-check output |
| `repair` (`repair.ts`) | resumes `build`'s session | fresh session: ticket + `plan.md` + `build.md` + the failing gate output + a bounded diff |
| `ci-repair` (`ci-round.ts`) | resumes the run's latest session | fresh session: ticket + `plan.md` + the failing CI check output + a bounded diff |

The bug lane's README rationale (*"a cold restart would throw away the context
in which the agent decided the test's shape"*) becomes an artifact
requirement: `build-test-only` must write down the test's intent in its
artifact, so `build-fix` reads it instead of inheriting a conversation.

## Requirements

- [x] **R1 — no cross-step `resume`** at the three sites above. A test per site
      asserts the agent call carries no `resume` option.
- [x] **R2 — each step's inputs are its artifacts**, as in the table. The
      prompts (`bug-build-fix.md`, `repair.md`, `review-fix.md` if it
      resumes, and the ci-repair prompt) gain the named inputs, with gate and
      CI output and diffs size-bounded by the existing truncation policy.
      *Amended 2026-09-27 (orchestrator): `repair` keeps its **untailed** `diffSoFar`, which is plan §6's anti-thrash anchor and binding over this ticket. Every other R2 input is met (PR #126).*
- [x] **R3 — `build-test-only` records the test intent** (what the test
      reproduces, and why it must fail on the base) in its artifact. A
      missing artifact fails `build-fix` fast, with zero agent calls.
- [x] **R4 — the persisted provider and model still win.** The resume sites
      also carried "resume on the same model" semantics (the attempt
      record's provider and model). A fresh repair or ci-repair session must
      still use the run's persisted provider and model. Test it.
- [x] **R5 — docs.** README "The bug lane" and the lane diagram comments in
      `justfile` stop saying `RESUME`. Replace them with the artifact handoff.
- [ ] **R6 — measure.** One live bug-lane run: record `build-fix`'s first-call
      context and time-to-first-edit in the PR body, next to perf-01's 126 s
      baseline for the resumed version.
      *Deferred 2026-09-27: needs a live bug-lane run on post-#126 code; to be recorded from the next bug-lane run (as perf-05 R7 was).*

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, plus R6's live run.

## Blocked by

- adw-perf-05-steps-are-isolated-and-the-plan-is-the-contract
