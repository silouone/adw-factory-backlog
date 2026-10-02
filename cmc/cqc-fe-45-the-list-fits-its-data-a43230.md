---
id: cqc-fe-45-the-list-fits-its-data-a43230
type: feat
status: in-progress
priority: 2
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: [{"runId":"cqc-fe-45-the-list-fits-its-data-a43230-1790968353628","branch":"adw/cqc-fe-45-the-list-fits-its-data-a43230","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-45-the-list-fits-its-data-a43230-1790968353628/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# The list fits its data

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 55–73 and decision **D-R2** (N/A stays visible, quietly).

## The problems (measured at 2000 px)

- Two widths: the paste bar and summary span the viewport; the table stops at ~1550 px, leaving a
  ragged edge and a 450 px hole.
- Summary (`CheckedContentList/CheckedContentSummary.tsx`): four large boxes for "3, 1, 0, N/A"
  under an extra "Backlog summary" h2.
- Filter labels are not on one baseline ("Search" ~10 px higher than the selects).
- Counts twice: "3 learning objects" above and "Showing 1–3 of 3" below.
- Table headers (`columns.tsx`) are uppercase and wrap ("AUTHORING / TOOL", "FIRST DETECTED · /
  LAST RUN", "DAYS / OPEN") despite free width.
- Content cell: the LO id is bold and the title regular; with no title the subtitle is
  "N/A · rev N/A".
- First detected and last run are identical on every row; Runs is almost always 1 and
  right-aligned far from its header; Env ("local", "E2B") is engineering detail.
- Actions column is ragged (one or two buttons per row) and the buttons look disabled.
- "Days open" shows "8 days" for Fail-block and "—" for Needs review with no explanation.

## The target

- One content width for the whole page (match the table to the band, or the band to the table;
  say which in the PR).
- Summary as **one stat band** without the extra h2; each stat is a button that applies the
  matching filter/tab (Open → Open tab, Fail-block → Status filter). Labels visible (fe-34).
  "Partners affected" stays and shows N/A quietly (D-R2).
- Filter labels share one baseline; the disabled Partner filter stays (D-R2) with its reason
  available, not as a floating ⓘ.
- One count: keep the pagination line, drop "3 learning objects" (or the reverse; one only).
- Headers in sentence case on one line: "Content", "Status", "Tool", "Connected", "Detected",
  "Open for", "Runs".
- Content cell: title first when known; `LO <id>` in mono, secondary. Missing title/partner/
  revision render as **quiet N/A** (subtle, `fontSize={1}`), never hidden.
- Detected cell: one date when first detected = last run; two lines only when they differ.
- Env moves to the drawer/page (Run facts). Runs is left-aligned with its header.
- Actions: a fixed-width cell; the primary row action is a real button, secondary actions in a
  consistent position (empty slot when absent, so the column aligns).
- "Open for" shows a value for every open case, or the reason it is not counted.

## Acceptance criteria

- [ ] Red tests first:
  - clicking the Fail-block stat sets the Status filter to Fail-block;
  - no header text is uppercase in the DOM text and none contains "·";
  - a row without a title renders "LO <id>" as its primary text and N/A as secondary;
  - a row whose first detected equals last run renders one date;
  - only one count line renders.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

At 1440 and 2000 px: no ragged right edge, headers on one line, rows ≤ 96 px.
