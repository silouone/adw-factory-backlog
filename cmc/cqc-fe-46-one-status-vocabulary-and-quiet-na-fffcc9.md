---
id: cqc-fe-46-one-status-vocabulary-and-quiet-na-fffcc9
type: chore
status: queued
priority: 2
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-38-the-journey-track-is-the-navigation-be1def, cqc-fe-39-the-header-says-the-verdict-in-one-sentence-c7d658, cqc-fe-40-checks-come-before-limitations-in-plain-words-192e22, cqc-fe-44-technical-values-are-mono-and-copyable-28f7e0, cqc-fe-45-the-list-fits-its-data-a43230]
attempts: []
---
# One status vocabulary, and N/A is quiet everywhere

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Items 74–77 and decision **D-R2**.

## The problem

Status words vary by place: "Fail-block" (pill), "failed" (check), "Failed" (strip), "not
verified" vs "Needs review" vs "unverified", "latest run not Pass" (code casing in prose). The
UI also explains itself ("They stay listed so the gap is visible…", "Most direct → least direct")
and shows checker "(s)" plurals. N/A renders at full body size in many places.

## The target

- One vocabulary, defined once in `domain.ts` (or the existing presentation map):
  - LO level: **Fail-block**, **Needs review**, **Pass**.
  - Check level: **Passed**, **Failed**, **Not verified**.
  Every label in the list, strip/track, checks, header and summary reads from it.
- A single quiet N/A renderer (e.g. `common/Missing.tsx`): `color="subtle"`, `fontSize={1}`,
  text "N/A". Replace user-visible uses of `formatMissing`'s raw string in JSX with it (keep
  `formatMissing` for non-JSX strings such as `title` attributes).
- Remove UI-explaining copy; no "(s)" plurals in user-facing strings.

## Acceptance criteria

- [ ] Red tests first:
  - no rendered text in ContentQuality matches `/not Pass|unverified|\(s\)/`;
  - the quiet N/A renderer is used for a missing title in the list and the header;
  - the check-level vocabulary is exactly Passed / Failed / Not verified across track and
    checks.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.
