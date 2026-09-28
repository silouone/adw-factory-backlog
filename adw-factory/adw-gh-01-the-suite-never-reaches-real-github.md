---
id: adw-gh-01-the-suite-never-reaches-real-github
type: bug
status: queued
priority: 1
created: 2026-09-28
caps: {minutes: 180, turns: 400, stallMinutes: 25}
depends: []
attempts: []
---
# A test run exhausted the operator's GitHub quota: nothing stops the suite from spawning real `gh`

Parent: `adw-gh`.

## The defect

On 2026-09-28 the in-flight `adw-pr-02` worktree added a real-`gh` default
(`defaultPrIndexGh`, `Bun.spawn(["gh", ...])`) behind
`WebServerDeps.prIndexGh`.

- Its own doc comment says the suite "must never" spawn real `gh`.
- But existing `startWebServer({...})` calls in `test/web/server.test.ts`
  don't inject the seam.
- So every test that started a server fired
  `gh pr list --repo <each target> --limit 300 --search head:adw/ --json …statusCheckRollup`
  against live GitHub.
- It ran on every gate and every agent-run `bun test`. The process tree
  showed `bun test --reporter=junit` as the `gh` parent.
- GraphQL hit 0/5000 and two other finished runs blocked at `open-pr`.

The rule "tests inject fakes" is enforced only by reviewer attention. One
missed seam in one ticket spends a quota every lane depends on.

## Fix

- The test preload (`test/setup/`, already loaded by `bun run test`) makes
  an un-injected `gh` fail **loudly and immediately**, for example:
  - put a stub `gh` first on `PATH` for the test process that exits non-zero
    with `adw test guard: real gh invoked: <argv>`
  - or an equivalent guard at the spawn seam
- Same guard for `git push` to a non-local remote if that's cheap. Otherwise
  note it as a follow-up.
- Red first: a test that calls a real-`gh` default without injection must
  fail with the guard message. A test that injects a fake must be
  unaffected.
- The guard must not break the few tests that intentionally exec a real
  local binary (`git` on fixture repos). Scope it to `gh`.

## Done when

- `bun run test` on a tree that re-introduces an un-injected `gh` default
  fails with the guard message, not a network call.
- Full suite green. `lint`, `tsc` green.
- If `adw-pr-02` lands first, its server tests pass only because they inject
  `prIndexGh`. The guard proves it.
