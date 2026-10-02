---
id: cqc-fe-38-the-journey-track-is-the-navigation-be1def
type: feat
status: queued
priority: 1
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-37-the-run-report-is-a-page-994cbb]
attempts: []
---
# The journey track is the navigation

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 14–18, 31, 68 and the "Signature" section.

## The problem

The check strip (`RunDrawer/CheckStrip.tsx`) and the tab bar (`RunDrawer/RunDrawerTabs.tsx`) are
two stacked navigation rows; users cannot tell which drives what. Strip labels wrap to two lines,
icons mix three styles (filled check, outlined "?", filled red cross), the selected cell is a pale
fill only, and nothing shows that the checks are a sequence. The list row repeats the six states as
six unlabelled 14 px icons under the status pill (`CheckedContentList/columns.tsx`).

## The target

The six checks really are an ordered journey (launch → navigate → complete → not forced → media
→ resume). Make that the signature:

- A **journey track**: six numbered stations joined by a line, each with one icon style, the
  check's short name on **one line**, and its state word below (Passed / Failed / Not verified).
  Failed stations are the strongest element; the selected station has a clear border/underline
  plus `aria-current`.
- On the **page** (fe-37) the track is the check navigation: selecting a station shows that
  check in the Checks tab (no separate "Show all checks" list header). Remove the Checks tab's
  count badge ("1 failed · 4 not verified"); the track already says it.
- In the **peek drawer**, the same track, non-interactive or linking to the page at that check.
- In the **list row**, a compact track (stations + line, no labels) with an accessible name per
  station ("Completion is recorded: failed").
- Keyboard: ← / → move between stations when the track has focus (roving tabindex).

## Acceptance criteria

- [ ] Red tests first:
  - the track renders six stations in journey order, each with a visible state word;
  - selecting station 3 shows "Completion is recorded" in the Checks tab and sets `aria-current`;
  - arrow keys move focus between stations;
  - the list row's compact track exposes six accessible names.
- [ ] The Checks tab no longer renders a count badge.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
