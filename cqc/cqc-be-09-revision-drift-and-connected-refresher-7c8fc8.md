---
id: cqc-be-09-revision-drift-and-connected-refresher-7c8fc8
type: feat
status: queued
priority: 2
created: 2026-09-28
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: []
---
# Each LO row shows its current revision, drift and whether it is connected (refresher)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Writers → Refresher" and the dev/staging note; BE-21/31, BE-33, BE-34,
BE-35, BE-37.

## What to build

A refresher run hourly (EventBridge) and for one LO after each projector write. For each `LO#`
with an `asset_portal_id` it lists `go1-scormassets/<env>/<asset_portal_id>/<lo>/`, and it
classifies the latest revision's files as connected (proxy-shell) or static, as a TypeScript
port of the existing SCORM classification script.

## Acceptance criteria

- [ ] `current_revision` = the max **numeric** prefix; non-numeric keys are ignored.
- [ ] `revision_changed` = `current_revision !== latest_run.go1_asset_revision` when both are known, else `null`.
- [ ] `connected` matches the shell script's verdict on the same listings (table-driven test).
- [ ] Missing grant or empty prefix → fields stay `null` and `{ lo_id, asset_portal_id, reason }` is logged; nothing throws.
- [ ] `prod` reads `production/`, `dev` and `staging` read `qa/`.

## Blocked by

- cqc-be-03-published-run-becomes-a-row-projector-125f20
