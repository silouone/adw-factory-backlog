---
id: cqc-be-10-openapi-single-contract-ba6824
type: chore
status: queued
priority: 3
created: 2026-09-28
review: false
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-04-list-checked-los-get-content-961b88, cqc-be-05-lo-history-and-run-report-e7a7ca, cqc-be-06-open-an-artefact-presigned-url-5ab9db]
attempts: []
---
# backend/openapi.yaml becomes the single API contract

Sources: `docs/backend/spec-cqc-backend-release-1.md` (canonical until this lands); BE-29; `docs/backend/todo.md` "First build steps".

## What to build

An OpenAPI document describing the release-1 endpoints **as built**, including the error shape
and the `RunSummary` / `ContentRow` schemas. The spec then points to it as canonical.

## Acceptance criteria

- [ ] Every release-1 endpoint, parameter, response and error code is described.
- [ ] The document validates (a lint or validator script in `backend/`).
- [ ] The spec says OpenAPI is now canonical; the CMC's `backend-contract.md` retirement is noted for the CMC side.

## Blocked by

- cqc-be-04-list-checked-los-get-content-961b88
- cqc-be-05-lo-history-and-run-report-e7a7ca
- cqc-be-06-open-an-artefact-presigned-url-5ab9db
