---
id: adw-m0
type: epic
status: done
priority: 1
created: 2026-07-14
children: [adw-m0-01-toolchain, adw-m0-02-claude-md]
attempts: []
---
# EPIC M0 — Bootstrap

Stand up the factory repo's toolchain from zero: Bun + strict TypeScript +
Biome + `bun test`, and the CLAUDE.md that binds every builder to the
constitution. No production code lands in this epic.

## Children

| Ticket | Scope |
|--------|-------|
| adw-m0-01-toolchain | package.json, strict tsconfig, biome, test wiring, gitignore |
| adw-m0-02-claude-md | minimal CLAUDE.md: constitution pointer + TDD/purity rules |

## Exit criteria

`bun install && bun run lint && bunx tsc --noEmit && bun test` all green on a
repo containing one trivial test and no production code.
