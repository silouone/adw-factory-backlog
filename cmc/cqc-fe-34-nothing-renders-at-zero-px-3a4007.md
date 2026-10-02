---
id: cqc-fe-34-nothing-renders-at-zero-px-3a4007
type: bug
status: in-review
priority: 1
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: [{"runId":"cqc-fe-34-nothing-renders-at-zero-px-3a4007-1790968330204","branch":"adw/cqc-fe-34-nothing-renders-at-zero-px-3a4007","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-34-nothing-renders-at-zero-px-3a4007-1790968330204/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"},{"runId":"cqc-fe-34-nothing-renders-at-zero-px-3a4007-1790968604411","branch":"adw/cqc-fe-34-nothing-renders-at-zero-px-3a4007-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-34-nothing-renders-at-zero-px-3a4007-1790968604411/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/169","provider":"codex","model":"gpt-6-sol"}]
---
# Nothing renders at 0 px

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 1, 78.

## The problem

Go1d's `type.scale[0]` resolves to **0 px** (measured in Chrome: `font-size: 0px`), so every
`<Text fontSize={0}>` is invisible while still present in the DOM:

- `CheckedContentList/CheckedContentSummary.tsx:60` — the summary card labels "Open",
  "Fail-block", "Revision changed", "Partners affected". The cards show "3 / latest run not Pass"
  with no title.
- `RunDrawer/CheckStrip.tsx:107` and `:112` — the strip's state words ("failed", "not verified").
  State is conveyed by icon alone: an accessibility failure.
- `RunDrawer/RunDrawerHeader.tsx:359` — the "Checker summary" label.

## The target

- Every one of these renders at `fontSize={1}` (the Go1d "secondary, metadata" size, see
  `.agents/design-system/foundations.md`), keeping their current colour and weight.
- Confirm the 0-index behaviour in the go1d source at the installed revision, and say so in the
  PR body (one line with the file and line of `type.scale`).

## Acceptance criteria

- [ ] Red tests first:
  - a source guard test fails while any file under `src/components/ContentQuality/` contains
    `fontSize={0}` (read the files with `fs`, excluding tests);
  - the summary renders each card label as visible text with the expected name;
  - each strip cell whose check is not Pass exposes its state word as text (not only an icon).
- [ ] No other visual change.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

The summary cards show their labels; the strip shows "failed" / "not verified" under each name.
