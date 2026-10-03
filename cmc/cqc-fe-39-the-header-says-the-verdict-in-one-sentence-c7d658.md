---
id: cqc-fe-39-the-header-says-the-verdict-in-one-sentence-c7d658
type: feat
status: in-progress
priority: 1
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-37-the-run-report-is-a-page-994cbb]
attempts: []
---
# The header says the verdict in one sentence

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 6–13, 21, 52, 53.

## The problem

`RunDrawer/RunDrawerHeader.tsx` stacks about 270 px in seven differently styled lines: breadcrumb,
"LO 10622213" title with the name in grey, verdict pill + "Limited evidence ⓘ" (reason hidden in
a tooltip), a dot-separated run-meta line, the checker's raw prose ("Observed 0
content-position transition(s) across 0 lesson(s)…") with "More", and a second dot-separated
case line starting with a large grey "Open". "Open in player" sits alone top-right while every
other action is in the footer. The "imported run" banner (`RunTruthBanners.tsx`) repeats at the
top of every tab.

## The target

```
Content quality › R U OK? (Hospitality)                 1 of 3  ↑ ↓  [j/k]  ✕
R U OK? (Hospitality)                [⧉ link] [Open in player] [Resolve…] [Run again]
LO 10622213 · Allara Learning · rev N/A          Open 8 days · first seen 23 Sep 2026
✕ Fail-block  Completion is never recorded.
  Limited evidence: preview launch, tracking off, desktop only. Imported run.
```

- **Title first** when Go1 returns one; ID in mono at secondary size. With no title, the ID is the
  heading (never "N/A" as a heading).
- **One verdict sentence** built from the failing check(s) (e.g. the first failing check's name
  as a sentence). The raw checker prose moves to the Technical tab ("Checker summary").
- **Limited evidence says why inline** (launch mode, tracking, devices tested), from the fields
  that drive the tooltip today; the tooltip goes.
- **Run facts and case facts** are two visually distinct groups, not two identical dot lines.
- **Imported run** is stated once in the header, not as a banner on every tab.
- **Actions together**: copy-link as an icon button by the title, then Open in player, Resolve…,
  Run check again. The j/k hint is `<kbd>` next to the ↑ ↓ buttons; the footer goes on the page
  (keep it in the peek drawer only if fe-37 kept actions there).
- Keep N/A per D-R2 (quiet, never hidden).

## Acceptance criteria

- [ ] Red tests first:
  - with a title, the heading is the title and the LO id is secondary; without one, the heading
    is "LO <id>";
  - the header renders a limited-evidence reason as text, and no tooltip trigger for it;
  - the "imported run" message renders once per report, not once per tab;
  - the checker prose is not in the header and is in the Technical tab.
- [ ] Header height at 1440 px ≤ 200 px (state the measured value in the PR).
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
