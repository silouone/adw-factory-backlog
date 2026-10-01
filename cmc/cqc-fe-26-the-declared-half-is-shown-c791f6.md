---
id: cqc-fe-26-the-declared-half-is-shown-c791f6
type: feat
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-26-the-declared-half-is-shown-c791f6-1790795123159","branch":"adw/cqc-fe-26-the-declared-half-is-shown-c791f6","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-26-the-declared-half-is-shown-c791f6-1790795123159/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# The drawer shows what the package declares, next to what the run observed

> **Reset to `queued` 2026-10-01.** The first attempt blocked on
> `ContentQuality > renders manual resolution outcome, person, date-time and note in the
> drawer`, which this ticket did not introduce: `cqc/release-1` was briefly red and the
> repair loop spent all three rounds on someone else's failure. Nothing was committed and
> no branch was pushed. The branch is green again (1028 tests, 123 suites), so start from a
> clean read of the ticket — do not try to work around that test.

> 2026-09-30: dispatched early and blocked on a false-green baseline; nothing to salvage. Requeue only after cqc-be-34 is done AND deployed to dev (cqc-be-34 is still queued).

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
