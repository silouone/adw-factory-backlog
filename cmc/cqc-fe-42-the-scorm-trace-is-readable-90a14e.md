---
id: cqc-fe-42-the-scorm-trace-is-readable-90a14e
type: feat
status: in-review
priority: 2
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-41-the-evidence-viewer-fills-the-height-20b912]
attempts: [{"runId":"cqc-fe-42-the-scorm-trace-is-readable-90a14e-1791027702729","branch":"adw/cqc-fe-42-the-scorm-trace-is-readable-90a14e-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-42-the-scorm-trace-is-readable-90a14e-1791027702729/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/178","provider":"codex","model":"gpt-6-sol","rebased":"c9e95d1a50165a0f32d305edb13e816e2096eb39"}]
---
# The SCORM trace is readable

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 39–42.

## The problem

`RunDrawer/Evidence/ScormTraceTable.tsx`: the TIME column shows raw values like `5366` with no
unit or origin; Element and Value show "N/A" for calls that take no element (Initialize, Commit,
GetLastError) — that is *not applicable*, not missing; "56 calls · 21 writes" but only the last
status write is highlighted and GetLastError noise drowns the writes; element names wrap so row
heights vary.

## The target

- Time as `+0.000s` relative to the first call (state the source unit in a column header tooltip-
  free caption, e.g. "Time since Initialize"). If the unit cannot be determined from the data
  contract, show the raw value with its field name and say so in the PR.
- Not-applicable cells are empty (with a visually hidden "not applicable" for screen readers).
  Genuinely missing values keep N/A per D-R2.
- Two toggles above the table: **Writes only** and **Hide GetLastError**, with the visible count
  updating ("21 of 56 calls").
- Every `SetValue` row is marked as a write (icon + text), not only the last status write; the
  last status write keeps its extra label.
- Mono, no wrap for Element and Value, fixed row height, horizontal scroll inside the viewer if
  needed. Tabular figures for Time.

## Acceptance criteria

- [ ] Red tests first:
  - Initialize renders no "N/A" in Element/Value;
  - "Writes only" leaves only SetValue/Commit rows (state which in the PR) and updates the count;
  - "Hide GetLastError" removes those rows;
  - the first row's time renders as `+0.000s`.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
