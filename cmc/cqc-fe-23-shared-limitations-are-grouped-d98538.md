---
id: cqc-fe-23-shared-limitations-are-grouped-d98538
type: feat
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: []
---
# Limitations shared by every run of a profile are grouped apart from this run's own

Sources: CMC spec "Shared limitations" (`spec-cqc-fe-release-1.md:262-266`),
`open-questions.md:32`; the prototype (`docs/cqc/prototype/drawers.js:204-220`). Backend:
`cqc-be-31` adds `limitations_meta: [{text, shared}]` to `GET /cqc/checks/{id}`.

The `cqc-be-*` blocker below lives in the CQC ticket store (`~/adw/backlog/cqc`), so it is **not** in `depends:`, because `just next` only resolves ids in the same store. Check that it is `done` (merged **and deployed to dev**) before dispatching.

## Today

`ChecksTab.tsx:49-76` renders `report.limitations` ungrouped, under "What this run could not
observe".

## Red first (Art. I)

`ChecksTab` test: with `limitations_meta` marking two entries as shared, those two render in a
collapsed "Shared by every run of this profile" group, and the rest stay listed.

## Acceptance criteria

- [ ] `CheckDetailResponse` gains `limitations_meta` (`ContentQuality.service.ts:292-297`).
- [ ] If `limitations_meta` is absent, the list stays ungrouped, as today.
- [ ] The text comes from `report.limitations`, and `limitations_meta` supplies only the flag.

## Blocked by

- cqc-be-31-limitations-say-when-they-are-shared-2fb535 (CQC store)
