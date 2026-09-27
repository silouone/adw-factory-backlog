---
id: adw-m0-01-toolchain
type: chore
status: done
priority: 1
created: 2026-07-14
epic: adw-m0
depends: []
attempts: []
---
# Toolchain scaffold

## Context

Empty repo (only `specs/`, `tickets/`, README). Everything the factory is
built with — Bun runtime, strict TS, Biome, `bun test` — starts here.

## Deliverables

- `package.json` — Bun project, **no workspaces**; scripts: `lint`
  (biome check), `typecheck` (`tsc --noEmit`), `test` (`bun test`)
- `tsconfig.json` — `strict: true`, `noUncheckedIndexedAccess: true`,
  `exactOptionalPropertyTypes: true`, `moduleResolution: "bundler"`,
  ESNext target/module, `types: ["bun-types"]`
- `biome.json` — lint + format; unused imports/variables are errors
- `test/smoke.test.ts` — one trivial passing `bun test` test
- `.gitignore` — add `runs/` (journal + spans live there, never committed)
  and `node_modules/`

## Requirements

- [x] No workspaces, no speculative deps — only `typescript`, `bun-types`,
      `@biomejs/biome` as dev deps at this stage (Art. II)
- [x] `runs/` gitignored before any run artifact can exist (was already
      present in `.gitignore`, verified) (plan §5 journal)
- [x] All four scripts runnable from a clean clone (N1: no API-key tooling,
      no paid services)

## Verify

```
bun install && bun run lint && bunx tsc --noEmit && bun test
```
All green; `bun test` reports exactly 1 passing test.

## Out of scope

Any `src/` code, CLAUDE.md (adw-m0-02), CI config.
