---
id: cqc-be-07-backfill-a-legacy-run-import-run-4fba2f
type: feat
status: in-progress
priority: 1
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-00-redaction-scan-without-side-effects-4de841, cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: [{"runId":"cqc-be-07-backfill-a-legacy-run-import-run-4fba2f-1790634981713","branch":"adw/cqc-be-07-backfill-a-legacy-run-import-run-4fba2f","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-07-backfill-a-legacy-run-import-run-4fba2f-1790634981713/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/19","provider":"codex","model":"gpt-5.6-sol"}]
---
# An engineer backfills a legacy run into a stage (import-run)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Writers → import-run", "Field mapping" (`captured_at`, `origin`); BE-5,
BE-8, BE-16, BE-17. Rules: `.agents/skills/cqc-guideline/` "One gate into a bucket", "Stage isolation".

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

A CLI in `backend/`: `import-run --stage <stage> <lo_id>/<run_id>` copies both of the run's
prefixes from the legacy bucket into the stage bucket, re-scanning every object with the shared
redaction module (from the redaction prefactor ticket).

## Acceptance criteria

- [ ] Every object passes the redaction re-scan; any surviving credential aborts the import before anything is written.
- [ ] `run-metadata.json` is uploaded last (it is the projector's commit marker).
- [ ] The legacy object's `LastModified` is recorded so the projector can use it as the last `captured_at` fallback, and the projected row carries `origin: "import"`.
- [ ] An existing destination prefix is never overwritten without `--force`.
- [ ] The legacy bucket is only ever read.
- [ ] Tests run against faked S3 ports; no test touches AWS.

## Blocked by

- cqc-be-00-redaction-scan-without-side-effects-4de841
- cqc-be-03-published-run-becomes-a-row-projector-125f20
