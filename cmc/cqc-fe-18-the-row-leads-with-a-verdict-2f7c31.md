---
id: cqc-fe-18-the-row-leads-with-a-verdict-2f7c31
type: feat
status: in-review
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-17-four-things-that-render-broken-b6e409]
attempts: [{"runId":"cqc-fe-18-the-row-leads-with-a-verdict-2f7c31-1790782779612","branch":"adw/cqc-fe-18-the-row-leads-with-a-verdict-2f7c31","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-18-the-row-leads-with-a-verdict-2f7c31-1790782779612/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/151","provider":"codex","model":"gpt-6-sol"}]
---
# A triage row leads with its verdict and fits on the screen

Audit: adw-factory `ai_docs/2026-09-30-cqc-frontend-ux-audit.md`, Parts 2 and 4.

## The job this page does

Internal content ops deciding **which learning objects need a human, and why** — at scale,
by scanning. Today the list presents seven columns of near-equal weight, repeats the same
six facts twice in one cell, and is 400px wider than the screen. Two rows fit.

## The three moves

### 1. Say the checks once

`CheckedContentList/columns.tsx:303-312` renders all six checks as a run-on sentence
(`"Content launches: passed · Navigation happens: not verified · …"`) and then the **same six**
as icons directly below. In a 240px column the sentence wraps to six lines.

Keep the icon strip; drop the sentence. The strip is the scannable unit — the ordered
journey `launch → navigate → complete → not-forced → media → resume`. Position carries
meaning because the checks are a sequence, so a user learns the shape once and reads it
everywhere. Give each icon an accessible name so the information is not lost, and let the
drawer carry the prose.

### 2. Collapse the facts

- `Timing` renders five stacked label/value pairs — ten lines. That is the
  information-display recipe, which is for detail views, used in a scanning context. Reduce
  to the last-run date, with days-open shown **only when the case is open**.
- `Package` (authoring tool, connection) becomes secondary text inside the `Content` cell,
  under the title — it describes the content, not a separate axis.
- `Case` and `Actions` collapse to one primary control plus a `MoreMenu`
  (`patterns.md` → Actions: *"Overflow or row actions: `MoreMenu` (`isButtonFilled={false}`
  in rows)"*, with `aria-label="Actions for <title>"` per `known-drift.md`).

### 3. Make the verdict the biggest thing in the row

The verdict pill is currently the same visual weight as everything else, in column two.
It is what the user is scanning for. Lead with it after the title.

## Targets

| | now | target |
| --- | --- | --- |
| columns | 7 | **5** |
| total width | 1720px + 128px gutter | **≤1180px** |
| row height | ~240px | **≤96px** |
| rows visible on a 1440×900 laptop | 2 | **6** |

## Acceptance criteria

- [ ] Five columns; the sum of the fixed widths is **≤1180px** and a test asserts it.
- [ ] No information is lost: everything the row shows today is either still in the row, in
      the row's accessible names, or in the drawer — say in the PR where each moved item went.
- [ ] The six-check strip has an accessible name per check carrying its status, so a screen
      reader still hears what the deleted sentence said.
- [ ] Row actions use `MoreMenu` with `aria-label="Actions for <title>"`.
- [ ] `—` / `N/A` handling still goes through `formatMissing` (CMC-SH-6 and the CQC
      placeholder policy); do not introduce a second rule.
- [ ] The table stays `GenericTable` (`.agents/conventions/references/recipes/table.md`);
      do not hand-build a table.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Explicitly not in scope

- The drawer (`cqc-fe-19`).
- Where the "Run checks" section sits (`cqc-fe-20`).
- Any change to what the API returns.

## Blocked by

- cqc-fe-17-four-things-that-render-broken-b6e409
