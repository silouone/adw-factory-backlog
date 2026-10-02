---
id: adw-perf-08-cli-suites-share-their-runs-9bac88
type: feat
status: in-progress
priority: 1
created: 2026-10-02
caps: {minutes: 150, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-perf-08-cli-suites-share-their-runs-9bac88-1790898806919","branch":"adw/adw-perf-08-cli-suites-share-their-runs-9bac88","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-08-cli-suites-share-their-runs-9bac88-1790898806919/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/176","provider":"claude","model":"claude-sonnet-5-5","rebased":"d1ecb7378c7844cf583be903b6d0648fc6356507"}]
---
# The cli.* suites run the same end-to-end ticket up to 11 times; one run should serve every assertion about it

Measured 2026-10-02 (`ai_docs/2026-10-02-test-suite-time-audit.md`, full suite under load):
the `test/cli*.test.ts` files make up 152 tests and 782 s out of a 1,923 s suite. Each test calls
`mkFixture` (a fresh `git init`, a commit, a bare origin and a push) and then a full `runTicket`
(about 55–65 git spawns). Many tests repeat **the same scenario** and differ only in what they assert.

| file | measured | duplicate runs found |
|---|---|---|
| `cli.test.ts` | 348 s / 49 | the plain chore green run is repeated 11 times: `:489, :520, :531, :552, :881, :906, :1004, :1145, :1841, :1855`, and the chore case of `:1820` |
| `cli.observability.test.ts` | 104 s / 15 | `:302, :483, :527, :558, :589, :706, :878` assert different fields of the same plain run; the enabled half of `:339` is that run too |
| `cli.skills.test.ts` | 97 s / 6 | three scenarios, each run twice (one test asserts the query options, the other the journal): `:244/:256, :277/:288, :308/:319` |
| `cli.provider.test.ts` | 83 s / 11 | `:254/:275` are the same scenario; `:234` duplicates `cli.test.ts:517` |
| `cli.systemprompt.test.ts` | 56 s / 4 | "absent" and "preset" expect the same result, because `DEFAULT_SYSTEM_PROMPT` is `"preset"` |
| `cli.allowedtools.test.ts` | 29 s / 4 | the absent case duplicates `agents.test.ts:377` |
| `cli-surface.test.ts` | 54 s / 90 | 65 `mkFixture` calls, but only 3 tests call `runTicket` |

In `cli.test.ts` some tests run the full green lane to check pure logic that already has a unit test:

- `:1401`, `:1519`, `:1540` check selection, covered by `select.test.ts:62,105`.
- `:1429` and `:1467` check depends, covered by `depends.test.ts:40-98`.
- `:989` checks the active-run refusal, covered by `cli-surface.test.ts:384`.
- `:1378` checks the manual type, covered by `ticket.test.ts:546`.

## What to build

- **One run, many assertions.** Run each repeated scenario once in `beforeAll` and keep its exit code, output, journal, query spy and git state. Each existing test asserts against that shared result. Tests that mutate the repo before or during the run keep their own fixture. In `cli.test.ts` those are `:927`, `:1128` and `:1285`. `:1004` deletes the workspace, so it runs last or keeps its own fixture.
- **Template fixture.** Build the git repo and bare origin once per file, then `cpSync` them per test. A copy needs no git spawn. The target JSON and origin URL are absolute paths, so rewrite them on each copy, or use a relative remote.
- **`remote: false` on the refusal tests.** None of these push, and the push is where the measured 30 s timeouts happened: `cli.test.ts:440, :456, :468, :989, :1093, :1123, :1345, :1378, :1429, :1467`.
- **Thin by coverage, not by count.**
  - Keep exactly one end-to-end smoke test for each concern: skills, systemPrompt, allowedTools and provider. Each asserts both the query options and the journal.
  - Turn the provider precedence (`:416, :436`) and the invalid value (`:398`) into `test.each` against an extracted pure `resolveProvider(flag, target)`. The logic is inline today at `src/cli.ts:359`.
  - Delete the selection, depends and manual tests listed above only after confirming the unit test named next to each one covers the same case.

## Red first (Art. I)

Add these before thinning anything:

- A unit test for `tracingDisabled(env)` (`src/cli.ts:95`). It has none today, and `cli.observability.test.ts:339` is its only cover.
- A unit test for `assertArtifactPathWritable` (`src/cli.ts:1471`). It has none; `cli.test.ts:1123` is its only cover.
- `test.each` for the new `resolveProvider`. This one is red until the function is extracted.
- A spawn budget: a test helper that counts `git` spawns for one file and fails over a stated ceiling. Measured today: `cli.test` 1,335, `cli.skills` 252, `cli.systemprompt` 168. Write the target ceilings into each file's header.

## Acceptance criteria

- [ ] Every behaviour asserted today is still asserted. The PR lists each deleted or merged test and the test that covers it now.
- [ ] Git spawns for the seven files at least halved, measured with the PATH-shim method from the audit, numbers in the PR.
- [ ] No `cli*` test needs more than the default timeout.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing). Related tickets: adw-perf-09 (the same method for node tests), adw-bug-39 (`cli.test.ts:594`).
