---
id: adw-bug-28-the-fix-stage-is-never-told-to-write-its-artifact-7f4283
type: bug
status: in-progress
priority: 1
created: 2026-09-27
review: false
caps: {minutes: 60, turns: 200}
depends: []
attempts: []
---
# The bug lane's fix stage is never told to write its artifact, so every bug run now blocks at build-fix

> Observed 2026-09-27 on `adw-gates-05-jest-failures-are-named-181b39-1790529943367`
> (the first bug-lane run after PR #126): `build-fix` ended with
> *"no artifact at .adw/artifacts/build-fix.md"* after the agent had implemented
> R1/R2, run the gates green and written a checkpoint marked `Status: done`.
> **Built live** (operator-mode precedent): the factory cannot fix its own bug
> lane through the bug lane.

## The defect

adw-perf-06 (#126) turned `build-fix` from a resumed session into a fresh session
that hands off through artifacts, and `build.ts` fails the node when
`.adw/artifacts/build-fix.md` is absent (adw-bug-07). But
`prompts/bug-build-fix.md` was never given the "Your written output" section
every other artifact-producing template carries (`chore-build`, `feature-build`,
`feature-test`, `feature-plan`, `bug-plan`, `bug-build-test`). The assembled
prompt's only mention of an artifact is the cap notice's *checkpoint* file, so
the agent writes the checkpoint and nothing else. `test/pipeline/templates.test.ts`
pinned the instruction for `bug-build-test.md` only.

## Requirements

- [x] **R1** `prompts/bug-build-fix.md` gains the same "Your written output —
      this is not optional" section, naming `.adw/artifacts/build-fix.md`, what
      the report must contain (files changed and why, root cause, requirements
      deliberately left out, gate results), the write-as-you-go rule, and that the
      checkpoint is not this report.
- [x] **R2** A red test in `templates.test.ts` pins the section for
      `bug-build-fix.md` (was RED: no such section).

## Verify

- [x] `bun test test/pipeline/templates.test.ts` red before, green after.
- [x] `test/pipeline/nodes/assemble-prompt.test.ts` and `test/pipeline/lanes` green.
- [ ] `bun run lint && bunx tsc --noEmit && just test-fast` green (running).
- [ ] The next bug-lane run's `build-fix` writes `build-fix.md` and reaches gates.
