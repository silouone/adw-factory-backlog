---
id: cqc-be-04-list-checked-los-get-content-961b88
type: feat
status: queued
priority: 1
created: 2026-09-28
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-02-staff-only-access-authorizer-2b1d03, cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: []
---
# Staff list every checked LO in the FE order (GET /cqc/content)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints" (`GET /cqc/content`), "Data model" (GSI `list`), `ContentRow`;
BE-4. The CMC page (`cqc-fe-03` onwards) is the consumer.

## What to build

`GET /cqc/content` reads the `list` GSI in order and applies the other filters in the handler
(fine up to ~2,000 LOs, per the spec).

## Acceptance criteria

- [ ] Order: Fail-block, failed_to_run, running, Needs-review, Pass; then `last_run_at` desc.
- [ ] Repeatable filters `status`, `environment`, `authoring_tool`, `connected`; plus `case_state` and `q` (matches an LO id or a run id).
- [ ] `offset` / `limit`, with `limit` capped at 100; `total` counts the filtered set.
- [ ] `summary` ignores the filters; `facets` reflect the available values.
- [ ] Response `{ items, total, offset, limit, summary, facets }`; `active_run` and `resolution` are always `null`.
- [ ] Bad query params → `400` with the shared error shape.

## Blocked by

- cqc-be-02-staff-only-access-authorizer-2b1d03
- cqc-be-03-published-run-becomes-a-row-projector-125f20
