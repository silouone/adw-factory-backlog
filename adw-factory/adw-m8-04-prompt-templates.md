---
id: adw-m8-04-prompt-templates
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m8-01-ticket-type-union, adw-m8-03-plan-test-agent-nodes]
attempts: []
---
# Six committed prompt templates + type/phase selection

> Source of truth: `specs/adw-v1.1-lanes.md` (Impl. decision 5; Graphs B & C).
> Read constitution + specs first. Same placeholder contract as
> `prompts/chore-build.md` + the new `{{plan}}` (m8-03).

## Context (grounded in source)

- `prompts/chore-build.md` — existing template; unchanged.
- `src/cli.ts:1231` — `promptTemplate` is a single string today; selection moves
  to where `ticket.type` (and bug phase) is known.
- Snapshot `test/pipeline/nodes/__snapshots__/assemble-prompt.test.ts.snap` —
  chore byte-identity guarantee; stays green **unchanged**.

## Requirements

- [ ] Six committed templates (N5), honoring the placeholder contract:
      - `feature-plan.md` — decompose the referenced spec into a plan artifact.
      - `feature-build.md` — **tests-first** implementation, consumes `{{plan}}`.
      - `feature-test.md` — expand coverage / edge cases on the built feature.
      - `bug-plan.md` — analyze the bug → repro-strategy plan artifact.
      - `bug-build-test.md` — write **only** a failing reproducing test, consumes
        `{{plan}}`; no fix.
      - `bug-build-fix.md` — (resume) the test is confirmed red; implement the
        minimal fix so it and all gates pass.
- [ ] Deterministic template selection by `ticket.type` (and bug phase) — a code
      switch seeded by the orchestrator; `assemblePrompt` signature unchanged.
- [ ] `chore` selects `chore-build.md`; snapshot byte-identical (no re-baseline).
- [ ] Every template passes the unknown-placeholder scan (only known placeholders
      incl. `{{plan}}` where applicable).

## Build protocol (red-first, Art. I)

1. Tests: selection returns the right template per type/phase; each renders
   without an unknown-placeholder throw; the chore snapshot is untouched. Confirm
   red, present for review.
2. Author templates + selection switch to green.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; chore snapshot identical;
selection + render tests pass.

## Out of scope

Lane composition (m8-06/07); node mechanics (m8-03).
