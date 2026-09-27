---
id: adw-profile-03-a-middle-gear
type: feat
status: done
priority: 1
created: 2026-09-18
review: false
caps: {minutes: 60, turns: 300}
depends: []
attempts: [{"runId":"adw-profile-03-a-middle-gear-1789762789951","branch":"adw/adw-profile-03-a-middle-gear","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-profile-03-a-middle-gear-1789762789951/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/81","provider":"claude","model":"sonnet"}]
---
# The operator has no middle gear — lowering effort also drops the model tier

> **Spec authority:** `specs/adw-v1.13-run-economics.md` §3 **S2**, §5 **D2**.
> Amends `specs/adw-v1.8-agent-profiles.md` §3.

## The problem

Measured across 50 banked runs: **every stage of every run runs at
`effort: high`**. `plan` averages 972 output tokens/turn, `review-standards`
1270 — extended-thinking numbers on stages that may not need them.

The only way to lower effort today is `cheap`, which *also* drops
`sonnet` → `haiku`. For `test` or `review-spec` that is too blunt to risk, so
nothing ever gets lowered.

`adw-v1.8-agent-profiles.md` §3 says `standard` is **medium**. The registry pins
it to **high**, deliberately: `adw-profile-01`'s R4 required an absent-config run
to emit options byte-identical to the pre-ticket `CHORE_MODEL="sonnet"` /
`CLAUDE_REASONING_EFFORT="high"`, and §4 resolves level 5 to *literally the
`standard` profile*. The medium tier disappeared as collateral from a
backward-compatibility guarantee.

## Requirements

- [ ] **R1 — a fourth profile.** `swift` = `(claude, sonnet, medium)`, added to
      the registry alongside `deep` / `standard` / `cheap`.
- [ ] **R2 — nothing else moves.** `standard` stays `(claude, sonnet, high)` and
      stays `FACTORY_DEFAULT_PROFILE`. An absent-config run must emit options
      byte-identical to today. v1.8 §4's level-5 rule is unchanged.
- [ ] **R3 — both validators accept it for free.** `agent: swift`,
      `agents: {test: swift}` and target `agents.feat.test = "swift"` all
      resolve. Both call sites already route through the registry's predicate —
      **confirm this rather than adding a branch.**
- [ ] **R4 — the module stays a leaf.** No imports from `intake/` or `targets/`
      (it is imported *by* both; a back-import is a cycle).
- [ ] **R5 — the spec table is corrected.** In
      `specs/adw-v1.8-agent-profiles.md` §3: add the `swift` row, and change
      `standard`'s effort cell from `medium` to `high` so the table matches the
      registry and the byte-identity requirement that forced it. Drop the
      in-source "diverges from the spec table" note, which no longer applies.

## Order of work — Article I

The existing test asserting the exact sorted key set of the registry
(`test/pipeline/profiles.test.ts`) **goes red the moment `swift` is added** —
that is your red test. Confirm it is red first, then implement.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] The registry key-set test is red before the change and green after, with
      the red output pasted into this ticket (Art. I).
- [ ] A test asserts `swift` is exactly `{provider:"claude", model:"sonnet", effort:"medium"}`.
- [ ] A test asserts `standard` is **unchanged** and `FACTORY_DEFAULT_PROFILE`
      still resolves to `standard`.
- [ ] A ticket carrying `agents: {test: swift}` parses; an unknown name is still
      rejected with a named error.
- [ ] `specs/adw-v1.8-agent-profiles.md` §3 has 4 rows and `standard` reads `high`.

## Out of scope

- Per-stage **provider** — that is `adw-profile-02`.
- Changing any model slug, or the factory default.
- Assigning `swift` to any stage. This ticket ships the gear; the operator
  shifts into it.
