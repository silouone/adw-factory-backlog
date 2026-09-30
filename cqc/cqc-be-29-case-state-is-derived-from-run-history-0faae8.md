---
id: cqc-be-29-case-state-is-derived-from-run-history-0faae8
type: feat
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: [{"runId":"cqc-be-29-case-state-is-derived-from-run-history-0faae8-1790794255651","branch":"adw/cqc-be-29-case-state-is-derived-from-run-history-0faae8","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-29-case-state-is-derived-from-run-history-0faae8-1790794255651/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/40","provider":"codex","model":"gpt-6-sol"}]
---
# `case_state` follows the run history: a Fail-block opens a case, and a later Pass resolves it

Sources: FE D-16 (case data is derived from run history), BE-38 (list semantics); CMC
`domain.ts:561-573` (which button shows).

## What's wrong today

`backend/src/common/run-transitions.ts:406-411` sets `open` only when a **new** LO's first run
is Fail-block. Otherwise it keeps the stored value. Nothing ever writes `resolved`: the only
non-test matches are reads in `get-content/listing.ts`. So:

- a Fail-block on an existing LO never opens a case;
- a Pass after a non-Pass never resolves it.

## Rule

- A succeeded run that lands as the **latest** run (`captured_at` order, `compareRun` :51-55)
  with verdict `Fail-block` sets `case_state = open`.
- The latest run `Pass` after an earlier non-Pass sets `case_state = resolved` (derived).
- A crashed or failed run (D-22) changes nothing.
- An out-of-order older run never overrides a newer one.
- This ticket does not touch a manual resolution; that is `cqc-be-30`. Leave the field alone.

## Red first (Art. I)

Pure `publishTransition(mapped, currentRun, currentLo)` table tests, in the style of
`run-transitions.test.ts:767`:

- existing LO `none`, then Fail-block → `open`;
- `open`, then Pass → `resolved`;
- an older out-of-order Fail-block after a newer Pass → unchanged.

## Acceptance criteria

- [ ] The three cases above, plus: a crashed run leaves `case_state` unchanged.
- [ ] The list filters (BE-38) and summary counts (`listing.ts:74-104`) reflect the new values
      without changing their code.
- [ ] Rows already stored are fine as they are; they update on their next transition. No
      backfill is needed. Say so in the PR.

## Blocked by

- (nothing)
