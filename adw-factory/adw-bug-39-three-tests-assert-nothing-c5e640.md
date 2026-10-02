---
id: adw-bug-39-three-tests-assert-nothing-c5e640
type: bug
status: in-review
priority: 2
created: 2026-10-02
depends: []
attempts: [{"runId":"adw-bug-39-three-tests-assert-nothing-c5e640-1790898820600","branch":"adw/adw-bug-39-three-tests-assert-nothing-c5e640","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-39-three-tests-assert-nothing-c5e640-1790898820600/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/172","provider":"claude","model":"claude-sonnet-5-5","rebased":"d1ecb7378c7844cf583be903b6d0648fc6356507"}]
---
# Three tests pass no matter what the code does

Found by the 2026-10-02 test audit (`ai_docs/2026-10-02-test-suite-time-audit.md` §4); each was checked by hand.

1. **`test/cli.test.ts:594`: a tautology.** `expect(spy.models).not.toContain("sonnet")`. `spy.models` holds full model IDs (`:291`), and the default is `"claude-sonnet-5-5"` (`:517`). `toContain` on an array matches whole elements, so this line can never fail. The intended check is "the ticket's `model: opus` replaced the default". Assert that `spy.models` does not contain the default model ID.
2. **`test/web/ui/usage-live.test.ts:242`: no assertion.** The test is "the returned stop function clears the interval". It calls `stop(); tick();` and asserts nothing. If `stop` did not clear the interval, `tick()` would just refetch and the test would still pass. Compare the `fakeFetch` call count before and after `tick()`.
3. **`test/codex-query.test.ts:633`: a dead runtime assertion.** The test calls itself a type-level guarantee, and `tsc` does enforce it. But its runtime `expect` sits inside the spawn `impl`. `void query(...)` never iterates the async generator (`src/codex-query.ts:612`), so `impl` never runs and the `expect` never executes. Either iterate the generator so the runtime check executes, or delete the dead `expect` so the test no longer reads as a runtime check.

## Red first (Art. I)

For 1 and 2, show that each corrected assertion fails against a deliberately broken implementation: the default model leaking through, or `stop` not clearing the interval. Record that in the PR. Then revert the break.

## Acceptance criteria

- [ ] Each of the three tests can now fail when the behaviour it names regresses, or, for 3, no longer pretends to check at runtime.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.
