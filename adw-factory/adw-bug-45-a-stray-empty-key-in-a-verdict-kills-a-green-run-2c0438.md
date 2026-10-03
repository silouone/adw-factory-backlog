---
attempts: [{"runId":"adw-bug-45-a-stray-empty-key-in-a-verdict-kills-a-green-run-2c0438-1791045758152","branch":"adw/adw-bug-45-a-stray-empty-key-in-a-verdict-kills-a-green-run-2c0438","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-45-a-stray-empty-key-in-a-verdict-kills-a-green-run-2c0438-1791045758152/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/204","provider":"claude","model":"claude-sonnet-5-5","stages":{"plan":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"build":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"test":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"build-test-only":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"build-fix":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"revise-test-only":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"repair":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"review-standards":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"review-spec":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"review-fix":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"ci-repair":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"rebase-resolve":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"}}}]
id: adw-bug-45-a-stray-empty-key-in-a-verdict-kills-a-green-run-2c0438
type: bug
status: in-review
priority: 1
created: 2026-10-03
depends: []
---
# A stray empty key in a reviewer's verdict kills a green run

> Spec: `specs/adw-v1.3-review-lane.md` §9 (the adw-bug-38 amendment). This
> ticket **amends §9** and must land the amendment with the code (Art. XI).

## Evidence

Run `adw-route-02-every-agent-node-is-routable-5d55f8-1791027763132` (67 min,
$4.50) was green on lint, typecheck, test and test-live, then ended `blocked` at
`review-standards`:

```
verdict parsed but invalid — findings[1].findings_note: unknown field "findings_note"
```

The raw verdict (`.adw/artifacts/review-standards.md`) is otherwise
well-formed: valid bare JSON, the right axis, and two `judgement` findings with
file, line, citation and summary. The only defect is one extra key whose value
is `""`:

```json
{"axis":"standards","findings_note":"","severity":"judgement","file":"src/pipeline/profiles.ts", ...}
```

Two `judgement` findings neither route nor block (v1.12 §3), so the run would
have opened its PR. Instead `review-spec` never ran, and 1312 lines sat
uncommitted until a hand salvage.

## Root cause

§9 classes an **unknown field** with inadmissible values as a terminal
contract violation: "the agent said something specific and wrong, and a re-ask
invites it to guess." That reasoning holds for an inadmissible value of a known
key (`severity: "critical"`). It does not hold for an unknown key: the parser
never reads it, so no routing decision depends on it, and naming it back
cannot make the agent invent one. `isRetryableParseFailure`
(src/pipeline/nodes/review.ts) treats both as `invalid`.

## Requirements

- [ ] **R1, a new error class.** `verdict.ts` gives an unknown key (top-level
      or per-finding) its own kind, `unknown`, distinct from `invalid`.
      Inadmissible values and axis mismatches stay `invalid` and stay terminal.
- [ ] **R2, re-asked, never dropped.** A verdict whose errors are only
      `envelope`, `missing` or `unknown` is re-asked within the shared
      `REVIEW_PARSE_MAX_ATTEMPTS` ceiling. The re-ask names each stray key path
      and says it is not part of the contract. The parser never silently
      strips it, and exhaustion still blocks.
- [ ] **R3, journaled.** `ParseRetry.class` gains `unknown` with the named
      fields, on the `review` node-end, exactly like `missing`.
- [ ] **R4, the amendment.** §9 of `adw-v1.3-review-lane.md` (both the repo
      copy and `~/adw/backlog/specs/adw-factory/`) records the split: an
      unknown key is a formatting slip, re-asked and never defaulted or dropped.
- [ ] A test pins the exact evidence verdict above: it is re-asked once, and a
      clean second answer proceeds.

## Out of scope

Tolerating unknown keys without a re-ask. Defaulting any field.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`
