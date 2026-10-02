---
id: hq-14-the-skills-deck-counts-match-the-graph-1aa94f
type: bug
status: in-progress
priority: 2
created: 2026-10-02
depends: []
attempts: []
---
# The skills deck counts match the graph's portable and claude-only totals

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Story 18, and an unmet hq-06 acceptance criterion ("tab counts equal the graph's totals"). `src/web/desk.tsx` hands the deck only `ofKind("skill")`, while the SKILLS ring and the library count commands and subagents too. On the live graph the deck shows 42 claude-only against the graph's 64, and 53 portable against 54. The hq-06 test passed only because its fixture had no commands or subagents.

## Red first

- A test whose fixture comes from `buildGraph` output, with at least one `command` and one `subagent` beside the skills, asserts each deck tab count equals the matching count over the graph's skills band.

## Acceptance criteria

- [ ] The deck, the SKILLS ring header and the library agree on one definition of "the skills band" (one shared selector, not three filters).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- (nothing)
