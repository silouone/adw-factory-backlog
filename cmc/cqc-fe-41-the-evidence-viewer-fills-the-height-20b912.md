---
id: cqc-fe-41-the-evidence-viewer-fills-the-height-20b912
type: feat
status: in-review
priority: 1
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-37-the-run-report-is-a-page-994cbb]
attempts: [{"runId":"cqc-fe-41-the-evidence-viewer-fills-the-height-20b912-1791011568019","branch":"adw/cqc-fe-41-the-evidence-viewer-fills-the-height-20b912","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-41-the-evidence-viewer-fills-the-height-20b912-1791011568019/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/173","provider":"codex","model":"gpt-6-sol"}]
---
# The evidence viewer fills the height

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 5, 33–38, 46. Operator screenshot: the snapshot viewer is a ~420 px box at the top
with empty space beneath.

## The problem

- Viewers are capped: `RunDrawer/Evidence/EvidenceViewer.tsx:173` and
  `SnapshotStepper.tsx:171` set `maxHeight: "320px"` with `overflow: auto`, so the trace and
  snapshots scroll **inside** the outer scroll (wheel trap, no keyboard focus target).
- Three notices stack before any evidence (~220 px): the imported-run banner, a grey "6
  limitations affect this verdict", and a yellow "1 of 4 cited items was not archived (⊘). They
  stay listed so the gap is visible…". A legend line ("Most direct → least direct · ⊘ = not
  archived") is needed to read the list.
- List items carry "Trust rank 2 of 8" chips (ranks 2, 6, 7, 7 for four items) and append the
  word "Selected" to the title; availability is plain text on some items and a chip on others.

## The target

- On the page (fe-37) the Evidence tab is a two-pane layout whose viewer pane **fills the
  remaining viewport height** (flex column down from the page, `min-height: 0` chain), with
  **one** scroll region: the viewer's own. Remove the 320 px caps. The list pane is sticky.
- At most one notice above the panes, and only when something is missing: "Screenshot not
  archived by this run." Limitations are linked from the Checks tab, not repeated here.
- The list order is the trust order; drop the rank chips and the legend. Each item: name, one
  line of description, and an availability word in one consistent style; unavailable items
  stay listed (gap visible) and say why when opened.
- Selection is `aria-current` + a visual state; no "Selected" text.
- The scroll region is focusable (`tabIndex=0`, labelled by the viewer heading).

## Acceptance criteria

- [ ] Red tests first:
  - no element in `Evidence/` sets a `maxHeight` in px;
  - the viewer's scroll region has `tabIndex=0` and an accessible name;
  - with one unavailable item, exactly one notice renders above the panes; with none, zero;
  - no list item renders "Trust rank" or "Selected" text.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

At 1440×900 and 2000×1200, the snapshot and trace viewers reach the bottom of the viewport;
only one scrollbar is visible in the Evidence tab.
