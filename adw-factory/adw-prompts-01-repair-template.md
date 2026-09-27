---
id: adw-prompts-01-repair-template
type: chore
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: []
---
# The repair prompt must be a committed template, not a string literal

Every agent stage renders its prompt from a versioned file under `prompts/`,
so tuning one is a reviewable diff. **Repair was the exception** — its prompt
was built by a `repairPrompt()` string literal in
`src/pipeline/nodes/repair.ts`.

That is the wrong place for it, for three reasons:

1. **It is the highest-pressure site in the factory.** Repair is the node
   running while gates are red, in every lane, for up to 3 rounds. It is the
   one prompt that needed an explicit anti-weakening prohibition
   (`adw-tier0-no-weakening`) — which tells you the pressure is real.
2. **Prompt changes there were code changes.** `prompts/` is where the
   operator's standards live; a repair-prompt tweak had to go through
   TypeScript instead.
3. **12-factor-agents Factor 2 ("own your prompts, treat them as first-class
   code").** We owned it, in the wrong place.

Surfaced by the prompt-system audit,
`ai_docs/2026-09-13-humanlayer-skills-human-on-the-loop.md` and
`ai_docs/prompt-system/04-repair-prompt.md`.

## Requirements

- `prompts/repair.md` is the committed template, carrying the full prompt text
  including the no-weakening prohibition.
- `repairPrompt(report, template)` renders it, rejecting an unknown
  `{{placeholder}}` before substitution — the same N5 discipline
  `assemble-prompt.ts` applies.
- The narrowing block (`introducedFailingTests`,
  `adw-auto-01-baseline-gate-snapshot`) renders into its own placeholder rather
  than being appended in code.
- The template is threaded through lane deps and read by the orchestrator, the
  `prBodyTemplate` precedent — not read from disk inside the node.
- **The rendered prompt is byte-identical to the string literal it replaces**,
  in both the block-absent and block-present branches.

## Verify

- `bun test test/pipeline/nodes/repair-template.test.ts` — red before, green
  after.
- `bun test test/pipeline/nodes/repair.test.ts` — the no-weakening tests now
  read the REAL committed template, so they pin the file the factory ships.
- `bun run lint && bunx tsc --noEmit && just test-fast` all green.
- Byte-identity checked against the pre-template literal for both branches.

## Done 2026-09-14

Implemented by hand (not dispatched), test-first. `prompts/repair.md` added;
`repairPrompt` now takes a template; `repairTemplate` threaded through
`ChoreLaneDeps`, `FeatLaneDeps`, `BugLaneDeps`, `CiRoundDeps`, `RunDeps` and
both shakedown scripts. `prompts/` now holds **8** agent-facing templates —
every prompt the factory sends is a file.
