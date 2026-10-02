---
id: cqc-fe-44-technical-values-are-mono-and-copyable-28f7e0
type: chore
status: in-progress
priority: 3
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-36-each-count-says-what-it-counts-e0dead]
attempts: []
---
# Technical values are mono and copyable

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 47–51.

## The problem

`RunDrawer/TechnicalTab.tsx` (the cleanest tab; keep its three-column grid): the git commit and
prompt hash wrap mid-string in a proportional font; "Working tree status: Uncommitted changes" is
the only coloured element (orange chip), so the eye lands on the least important fact; numbers
are left-aligned without tabular figures.

## The target

- Hashes and ids: mono, truncated to 12 characters with a copy button; the full value is the
  copy payload and the accessible name. No mid-string wrap.
- Working tree status: plain text like its neighbours (a warning icon + text is acceptable; no
  filled chip).
- Cost, duration and counts use tabular figures.
- Receive the "Checker summary" prose from fe-39 if it has landed (otherwise leave it).
- Keep "Not recorded for this run: …".

## Acceptance criteria

- [ ] Red tests first: the commit renders 12 characters and a copy button whose payload is the
      full hash; Working tree status renders no Pill/chip.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
