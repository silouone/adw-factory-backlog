---
id: adw-ci-workflow
type: chore
status: done
priority: 1
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-ci-workflow-1789129146156","branch":"adw/adw-ci-workflow","workspace":"/Users/silouane/adw-factory/runs/adw-ci-workflow-1789129146156/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/4","provider":"claude","model":"sonnet"}]
---
# CI: nothing has ever run this suite automatically

## Context

`.github/workflows/` does not exist. Nothing runs `bun test` unless a human or
the factory does. Consequence, measured 2026-09-11: two tests went over budget
on 2026-07-15 and 2026-07-19 and the suite stayed red for **two months**
without anyone noticing. The m8 work ran targeted tests; the full suite was
effectively unobserved.

The one mechanism that DID run it — the factory's own `gates` node — reported
the red as `target configuration error, not a gate failure`
(`adw-gates-config-error-regex`). Every safety net had a hole in the same place.

## Two reasons, and the second is the non-obvious one

1. **Catch suite rot.** A green `main` should be a fact, not a belief.
2. **Dogfood the factory's own CI-round machinery.** `ci-round.ts` polls
   `gh pr checks` — the whole adw-m2-04 / m4-04 / m5-03 resume-on-CI-red path.
   The self-target has **no checks**, so that subsystem has never run against
   adw-factory. Adding CI gives it something real to poll, and would have
   exercised it on PR #1.

## Requirements

- [ ] `.github/workflows/` running `bun run lint`, `bunx tsc --noEmit`,
      `bun run test` on push and PR. Use `bun run test`, NOT bare `bun test` —
      the 30s budget lives in the package.json script (commit `bfa753f`).
- [ ] Docker- and E2B-dependent suites must SKIP cleanly on the runner, not
      fail. `docker-gate.ts` / `e2b-gate.ts` already do this by probing for the
      dependency; confirm the runner trips them rather than half-running.
      `ADW_CODEX_SMOKE` stays unset, so Part H skips.
- [ ] No secrets required for a green run. If a job needs `E2B_API_KEY` or a
      Claude token it is the wrong job — those paths are operator-gated.

## Verify

- A PR against this repo shows checks, and `gh pr checks` returns them.
- A deliberately broken test fails the workflow.
- A runner without docker still goes green (suites skip loudly).

## Out of scope

Making CI a merge requirement — decide after it has run for a while.

## Note on what CI would and would not have caught

It would have caught the two genuinely over-budget tests. It would **not** have
caught the load-sensitive population, because a CI runner is quiet while the
operator's workstation runs other projects' container stacks. The 30s default
is what makes the suite survive a real machine; CI is what makes the red
visible. They are different fixes for different halves of the same failure.
