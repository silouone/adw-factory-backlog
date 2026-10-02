---
id: cqc-fe-36-each-count-says-what-it-counts-e0dead
type: chore
status: in-progress
priority: 2
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: [{"runId":"cqc-fe-36-each-count-says-what-it-counts-e0dead-1790968337623","branch":"adw/cqc-fe-36-each-count-says-what-it-counts-e0dead","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-36-each-count-says-what-it-counts-e0dead-1790968337623/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# Each count says what it counts

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Item 3.

## The problem

The Technical tab shows **"Snapshot count 36"** (Extraction section, `RunDrawer/TechnicalTab.tsx`)
while the Evidence tab's accessibility stepper shows **"Snapshot 9 of 52"**
(`RunDrawer/Evidence/SnapshotStepper.tsx`). On LO 10622213 both appear for the same run. They
are very likely different things (extraction snapshots vs archived accessibility `.yml` files),
but the same word makes them read as a contradiction. Likewise "Positions observed" (Extraction)
and "Positions" (Navigation) both read 0 and look duplicated.

## The target

- Read the run metadata fields behind each value (`domain.ts`, the contract types) and name each
  count after its source, e.g. "Text snapshots extracted" vs "Accessibility snapshots archived".
- If the two fields are in fact the same quantity and disagree, do not paper over it: render both,
  and say so in the PR body so the backend can be ticketed.
- Rename "Positions observed" / "Positions" the same way, or merge them if they are one field.

## Acceptance criteria

- [ ] Red tests first: the Technical tab never renders the bare label "Snapshot count"; the
      stepper's heading names the snapshot kind.
- [ ] The PR body lists each renamed label with the field it reads.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
