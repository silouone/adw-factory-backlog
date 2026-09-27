---
id: adw-m1-07-target-config-prompt
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-01-ticket-contract]
attempts: []
---
# Target config loader + assemble-prompt node

## Context

The factory must support any target repo via thin config — nothing
cLens-specific in the pipeline (N4). The build prompt is assembled
deterministically (S1.4) as a *light context pack*: ticket body +
conventions + **pointers** to context docs, contents not inlined
(plan decision 13). Prompts are versioned files (N5).

## Deliverables

- `test/targets/loader.test.ts`, `test/pipeline/nodes/assemble-prompt.test.ts`
  (first, red)
- `src/targets/loader.ts` · `targets/clens.json` (values from plan §5)
- `src/pipeline/nodes/assemble-prompt.ts` · `prompts/chore-build.md`

## Requirements

- [x] Loader validates `targets/<name>.json` against the plan §5 shape
      (`name`, `repo`, `base`, `branchPrefix`, `gates[{name,cmd}]`, `setup`,
      `context[]`, `ci{provider,poll}`); misconfiguration → descriptive
      rejection listing every bad field (N4, E3 flavor)
- [x] `assemblePrompt(ticket, target, template)` is pure and deterministic:
      same inputs → byte-identical prompt (S1.4); snapshot-tested
- [x] Prompt contains: full ticket body, target conventions (branch/gate
      expectations), and context doc **paths** for the agent to read itself
      (decision 13)
- [x] Template `prompts/chore-build.md` is a committed, versioned file with
      explicit placeholders — prompt tuning is a reviewable diff (N5)

## Build protocol (Art. I)

1. Loader tests (valid clens.json fixture; each field invalid; multi-error
   report). Prompt tests (snapshot; determinism; pointers-not-contents).
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

PR body template (adw-m2-02), grep-based context prefetch (spec §7 deferral).
