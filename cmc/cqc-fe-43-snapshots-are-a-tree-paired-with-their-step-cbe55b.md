---
id: cqc-fe-43-snapshots-are-a-tree-paired-with-their-step-cbe55b
type: feat
status: queued
priority: 2
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-41-the-evidence-viewer-fills-the-height-20b912]
attempts: []
---
# Snapshots are a tree, paired with their step

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 4, 43–45. Operator screenshot: raw YAML wraps and loses its indentation.

## The problem

`RunDrawer/Evidence/SnapshotStepper.tsx` renders accessibility snapshots (`runs/snapshots/step-NNNN.yml`)
as raw text in a proportional font. Wrapping destroys the tree indentation; icon-font
characters render as tofu (`tab "□ PHYSICAL APPEARANCE"`); `[ref=f5e52]` markers dominate; "Snapshot
9 of 52" has no link to a journey step or screenshot; Previous/Next are large buttons under a
fixed-height box.

## The target

- Mono, `white-space: pre`, no wrapping; horizontal scroll inside the viewer (fe-41 gives it the
  height).
- Parse the snapshot into lines with depth; render it as a **collapsible tree** (each node
  toggles its children; depth ≥ 4 starts collapsed). `[ref=…]` and `[cursor=…]` are dimmed
  (subtle colour), never removed.
- Private-use code points (U+E000–U+F8FF) render as a visible `[icon]` token, not tofu.
- Pair each snapshot with its step: if the run has a step index (as fe-25's stepper does for
  screenshots), show the step number and its screenshot thumbnail beside the tree.
- Navigation: a compact step scrubber (numbers or a range) plus ← / → keys when the viewer has
  focus; remove the two big buttons.

## Acceptance criteria

- [ ] Red tests first:
  - a fixture line containing U+E900 renders `[icon]`;
  - a node toggle hides and shows its children;
  - `[ref=f5e52]` stays in the text (dimmed);
  - → moves from snapshot 9 to 10 when the viewer has focus.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
