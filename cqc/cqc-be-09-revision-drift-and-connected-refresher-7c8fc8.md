---
id: cqc-be-09-revision-drift-and-connected-refresher-7c8fc8
type: feat
status: in-progress
priority: 2
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: [{"runId":"cqc-be-09-revision-drift-and-connected-refresher-7c8fc8-1790642844452","branch":"adw/cqc-be-09-revision-drift-and-connected-refresher-7c8fc8","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-09-revision-drift-and-connected-refresher-7c8fc8-1790642844452/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/23","provider":"codex","model":"gpt-6-sol"}]
---
# Each LO row shows its current revision, drift and whether it is connected (refresher)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Writers → Refresher" and the dev/staging note; BE-21/31, BE-33, BE-34,
BE-35, BE-37.

## Working in `backend/` (read first)

- The scaffold (PR #13) already holds every release-1 dependency and the lockfile: io-ts, fp-ts,
  AWS SDK v3 (DynamoDB, lib-dynamodb, S3, s3-request-presigner, SSM), Vitest + coverage, Biome,
  Serverless 3 + serverless-esbuild, aws-sdk-client-mock, tsx. **Do not add, remove or upgrade a
  dependency and do not run `npm install`**: the sandbox has no network. If something is truly
  missing, stop and say so in your final message.
- Reuse what cqc-be-01 built: `src/common/{errors,decoders,resource-names}.ts`, `utils/logger.ts`,
  the shared error shape, the `createPorts` pattern, and the `serverless.yml` conventions.
- Gates (in `backend/`): `typecheck`, `lint`, `test` with a **100% coverage threshold**, `config:check`.
  Follow `.agents/skills/cqc-guideline/` (`SKILL.md`, `references/typescript.md`).

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
