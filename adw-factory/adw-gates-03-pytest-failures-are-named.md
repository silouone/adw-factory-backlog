---
id: adw-gates-03-pytest-failures-are-named
type: bug
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 90, turns: 400}
depends: []
attempts: [{"runId":"adw-gates-03-pytest-failures-are-named-1789842022445","branch":"adw/adw-gates-03-pytest-failures-are-named","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-03-pytest-failures-are-named-1789842022445/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/91","provider":"claude","model":"sonnet"}]
---
# A pytest gate that fails names no failing test, so the baseline cannot subtract and repair chases the wrong thing

> Observed 2026-09-19 on `sabado-00-…-1789823904374`: four `test` gate
> failures, every one journaled `failingTests: []`, three repair rounds spent
> rewriting a test that passed, while `tests/test_admin.py::…` failed each
> time. `review: false`: hard-gated.

## The defect

`src/pipeline/nodes/gates.ts` `parseBunFailingTests` recognises only bun's
`(fail) <name>` lines. A pytest gate prints
`FAILED tests/test_admin.py::test_admin_disabled_when_token_unset - AssertionError…`
and `ERROR tests/x.py::y` lines in its short summary; both yield `[]`. With
`[]`, `classifyGateOutcome` cannot partition pre-existing from introduced
(adw-auto-01) and the repair prompt names nothing. The repair prompt's
2 000-character tail then decides what the agent sees, and on a two-runner
gate that tail is the second runner's.

## Requirements

- [ ] **R1** A pure `parsePytestFailingTests(stdout)`: every
      `^(FAILED|ERROR) (\S+::\S+)` line, in order, node id only (the
      ` - reason` suffix stripped), deduplicated.
- [ ] **R2** `parseFailingTests(stdout)` = bun ∪ pytest, order preserved;
      every caller of `parseBunFailingTests` uses it (gates, baseline,
      red-check). No behaviour change for bun output.
- [ ] **R3** `TAIL_CHARS` becomes 6 000 for the repair prompt; the journal
      keeps 2 000 (the journal is an index, the prompt sidecar is the body).

## Verify

- [ ] Red test: a pytest-shaped stdout fixture with two `FAILED` and one
      `ERROR` line → those three node ids, in order. RED today (`[]`).
- [ ] Red test: a bun fixture and a pytest fixture concatenated → both sets.
- [ ] Red test: the gates node, given the sabado-00 stdout shape (pytest
      summary then 3 400 vitest lines), journals the two admin test ids in
      `failingTests`. RED today.
- [ ] Existing gates/baseline/red-check suites green unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
