---
id: cqc-be-10-openapi-single-contract-ba6824
type: chore
status: in-progress
priority: 3
created: 2026-09-28
model: gpt-5.6-sol
review: false
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-04-list-checked-los-get-content-961b88, cqc-be-05-lo-history-and-run-report-e7a7ca, cqc-be-06-open-an-artefact-presigned-url-5ab9db]
attempts: []
---
# backend/openapi.yaml becomes the single API contract

Sources: `docs/backend/spec-cqc-backend-release-1.md` (canonical until this lands); BE-29; `docs/backend/todo.md` "First build steps".

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
