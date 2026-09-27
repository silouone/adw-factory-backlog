---
id: adw-tier0-no-weakening
type: feat
status: done
priority: 1
created: 2026-09-10
depends: []
attempts: []
---
# No-weakening rule in the repair prompt and the build templates

## Context

`tickets/BACKLOG.md` candidate 1. Promoted to a ticket 2026-09-10 because it
became a live blocker, not a nice-to-have: `main`'s suite is not
deterministically green (one reproducible 5 s timeout in
`test/cli.observability.test.ts`, plus docker/E2B live suites that flake under
load). A markdown-only ticket can therefore hit a **red gate it did not
cause** — and `repairPrompt()` (`src/pipeline/nodes/repair.ts:71`) says only
*"Fix the workspace so ALL gates pass, then stop."*

Repair is the highest-pressure site in the factory: every lane, up to 3 rounds,
and a weakened assertion is a legitimate-looking route to green that the gates
cannot distinguish from an earned one. SWE-Bench restores original tests before
scoring *because agents are known to comment them out* — a documented trained-in
behavior, not hygiene.

`prompts/feature-test.md:34` and `prompts/bug-build-fix.md:28` already carry the
prohibition. `repairPrompt()`, `prompts/chore-build.md` and
`prompts/feature-build.md` do not. This closes that gap.

## Deliverables

- `src/pipeline/nodes/repair.ts` — one line in `repairPrompt()`.
- `prompts/chore-build.md`, `prompts/feature-build.md` — one line each.
- Tests covering all three.

## Requirements

- [x] `repairPrompt()` output contains an explicit prohibition on weakening,
      skipping, deleting or commenting out a test to reach green.
- [x] The prohibition names the honest alternative: if the failing test is
      genuinely wrong or unrelated to the change, STOP and say so rather than
      edit it. A repair round that cannot honestly pass must fail loudly.
- [x] `prompts/chore-build.md` and `prompts/feature-build.md` carry the same
      prohibition, so it binds the FIRST agent turn and not only repair.
- [x] `repairPrompt` is exported so the rule is assertable without driving a
      whole engine run.

## Verify

- `bun test test/pipeline/nodes/repair.test.ts test/pipeline/templates.test.ts`
- `bun run lint && bunx tsc --noEmit`
- The new tests fail on the parent commit and pass after (Art. I).

## Out of scope

- The post-hoc test-validity check for the feat lane (BACKLOG candidate 2).
- Any change to gate execution, retry counts or lane graphs.
