---
id: cqc-be-03-published-run-becomes-a-row-projector-125f20
type: feat
status: queued
priority: 1
created: 2026-09-28
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-01-backend-skeleton-and-config-endpoint-199374]
attempts: []
---
# A published run becomes a Run row and updates its LO row (projector)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Data model", "Writers → Projector", "Field mapping", "Types"; BE-10,
BE-23, BE-28, BE-30, BE-31b, BE-32. Rules: `.agents/skills/cqc-guideline/` ("Observe, never fabricate", "Honest lifecycle",
"Pass-through contracts").

## What to build

- The stage table (`RUN#`, `LO#`, GSIs `lo-runs` and `list`) and the S3 `ObjectCreated`
  trigger on `navigation_artefacts/output/*/*/run-metadata.json`.
- The projector: read and decode `run-metadata.json`, look for the run's
  `playability-report.json` and validate it with the existing V1 validator, write `RUN#`
  with the spec's field mapping, then recompute `LO#` from all of that LO's runs.

## Acceptance criteria

- [ ] Every row of the spec's field-mapping table has a test, including `package` renamed to `{ vault_uuid, go1_asset_revision }` and `asset_portal_id` from `launch.configuration`.
- [ ] `captured_at` falls back in order: the field → the UTC stamp in `run_id` → the legacy `LastModified` recorded at import; never the stage bucket's `LastModified`; never null.
- [ ] Valid report → `succeeded`; no report → `failed` / `report_missing`; invalid report → `failed` / `report_invalid: <message>`. Nothing else is ever written in release 1.
- [ ] `LO#` holds the latest succeeded run, the last failed run, `first_detected_at`, `last_run_at`, `run_count`, and a `sort_key` giving the FE order (Fail-block, failed to run, running, Needs-review, Pass; then `last_run_at` desc).
- [ ] Replaying the same event rewrites identical items (idempotence test).
- [ ] An undecodable `run-metadata.json` fails with a named error naming the key.

## Blocked by

- cqc-be-01-backend-skeleton-and-config-endpoint-199374
