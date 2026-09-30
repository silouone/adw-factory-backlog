---
id: cqc-fe-26-the-declared-half-is-shown-c791f6
type: feat
status: queued
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: []
---
# The drawer shows what the package declares, next to what the run observed

Sources: FE D-24 (it hid the declared half until the backend could fill it). Backend:
`cqc-be-34` fills `package_extraction_snapshot` from `imsmanifest.xml`.

The `cqc-be-*` blocker below lives in the CQC ticket store (`~/adw/backlog/cqc`), so it is **not** in `depends:`, because `just next` only resolves ids in the same store. Check that it is `done` (merged **and deployed to dev**) before dispatching.

## Today

- `PlayabilityReport` has no `package_extraction_snapshot`
  (`ContentQuality.service.ts:175-200`).
- Each check's `declared` already renders (`RunDrawer/ChecksTab.tsx:144-159`).

## Red first (Art. I)

A drawer test: a report with a filled snapshot (SCORM version, SCO count, launch file,
mastery score, declared media) renders a "Declared by the package" block. A report with the
empty placeholder shows nothing.

## Acceptance criteria

- [ ] The type is added, and the block is shown only when the snapshot is non-empty.
- [ ] A declared-vs-observed mismatch is visible (e.g. declared media not reached).

## Blocked by

- cqc-be-34-the-report-declares-what-the-package-says-c8ab5f (CQC store)
