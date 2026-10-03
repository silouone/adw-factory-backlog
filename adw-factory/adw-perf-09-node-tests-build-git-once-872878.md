---
id: adw-perf-09-node-tests-build-git-once-872878
type: feat
status: in-progress
priority: 2
created: 2026-10-02
caps: {minutes: 150, turns: 400, stallMinutes: 25}
depends: [adw-bug-36-rebase-tests-time-out-under-load-538e90]
attempts: []
---
# The node tests rebuild real git repos per test, including where git is incidental to what they assert

Measured 2026-10-02 (`ai_docs/2026-10-02-test-suite-time-audit.md`, full suite under load):

| file | measured | git spawns (isolated run) |
|---|---|---|
| `test/pipeline/nodes/ci-round.test.ts` | 192 s / 84 | 1,208 |
| `test/pipeline/nodes/open-pr.test.ts` | 50 s / 116 | 323 |
| `test/intake/repo-commit.test.ts` | 45 s / 37 | (not run in isolation) |
| `test/pipeline/nodes/sync-pr-state.test.ts` | 37 s / 94 | (not run in isolation) |
| `test/workspace/e2b.test.ts` | 56 s / 48 | E2B itself is faked; all of the cost is 23 host-git `fixtureRepo()` builds |

adw-bug-36 covers `rebase.test.ts` and `rebase-round.test.ts`. This ticket builds on the fixture helper that bug-36 lands. It does not build a second one.

## Findings this ticket acts on

- **`test/real-git-workspace.ts:23`** runs every command through `sh -c`, so each git call costs two processes. Run git by argv.
- **ci-round, shared-run candidates.** About 12 tests use the identical `makeFixture()`, `fakeGh([CHECKS_FAILING])` and the default `fakeQuery`, and differ only in what they assert: `:618, :662, :816, :859, :939, :1020, :1895, :2071, :2158`, and the default case of `:2563`. The skills tests `:2121` and `:2170` pass the same `skills` value.
- **ci-round, incidental git.** About 25 tests use real git but only assert what reached the agent or the journal:
  - system prompt `:938/:958`, provider `:991-1047`, resume model `:1142/:1164`;
  - prompt threading `:751-816`, hooks and tracing `:1738-1894`;
  - skills and tools `:2071-2170`, codex `:1219-1501`, no-progress `:2935`, traceparent `:2995`.

  Give them the plain `TicketStore` seam instead.
- **open-pr, incidental git.** These tests only check gh argv or the retry result: `:2197-2335` and `:2677-2824`.
- **sync-pr-state, incidental git.** These tests only assert that nothing was written: `:700, :722, :747`, the dryRun group `:852-985`, and the malformed sweep `:807`.
- **repo-commit.**
  - About 16 tests inject a fully scripted `gitRunner` but still `git init` a repo they never use as git: `:598, :645, :689`, the `test.each` at `:873` and `:919`, and `:953`. A plain directory with `tickets/<id>.md` is enough.
  - The concurrency test at `:1086` runs 3 rounds, each with a new repo and 2 `bun run` children.

## What to build

- Add `sh -c` removal and a copy-from-template mode to the shared real-git fixture helper (the bug-36 helper). Use it in ci-round and sync-pr-state.
- Move the incidental-git tests listed above onto the plain store or a plain directory.
- Run the ci-round shared-run group once in `beforeAll` and assert each test against it.
- Run the repo-commit concurrency test with one repo across its rounds. Keep the round count only if the bug it guards needs it, and say why in the file header.

## Red first (Art. I)

- A spawn-budget assertion per file, with a stated ceiling measured on today's numbers. It is red until the work lands.
- Any test moved off real git first gets a sibling assertion, proven to fail on a broken implementation, that still exercises the property it covered.

## Acceptance criteria

- [ ] Same behaviours covered. The PR maps every moved or merged test to the test that covers it now.
- [ ] Git spawns for ci-round, open-pr, sync-pr-state and repo-commit at least halved (PATH-shim measurement in the PR).
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- adw-bug-36, which lands the fixture helper this ticket extends.
