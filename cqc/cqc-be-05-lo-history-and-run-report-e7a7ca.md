---
id: cqc-be-05-lo-history-and-run-report-e7a7ca
type: feat
status: queued
priority: 1
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-02-staff-only-access-authorizer-2b1d03, cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: []
---
# Staff open one LO's run history and one run's full report

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints" (`GET /cqc/content/{lo_id}/runs`, `GET /cqc/checks/{run_id}`);
BE-24a, BE-31b. Rules: `.agents/skills/cqc-guideline/` "Pass-through contracts".

## Acceptance criteria

- [ ] `GET /cqc/content/{lo_id}/runs` returns `{ items: RunSummary[] }` newest first (GSI `lo-runs`); an LO with no run → `404 lo_not_found`.
- [ ] `GET /cqc/checks/{run_id}` returns `{ run, metadata, report, artefacts }`; `metadata` and `report` are the S3 documents **byte-identical** (test compares bytes), `null` when absent.
- [ ] `artefacts` is `[{ name, content_type }]` from a listing of the run's prefixes.
- [ ] Unknown run → `404` with the shared error shape.

## Blocked by

- cqc-be-02-staff-only-access-authorizer-2b1d03
- cqc-be-03-published-run-becomes-a-row-projector-125f20
