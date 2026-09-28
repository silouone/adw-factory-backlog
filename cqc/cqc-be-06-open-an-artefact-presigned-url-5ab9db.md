---
id: cqc-be-06-open-an-artefact-presigned-url-5ab9db
type: feat
status: queued
priority: 2
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-05-lo-history-and-run-report-e7a7ca]
attempts: []
---
# Staff open a run's artefact through a short-lived link

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints" (`GET /cqc/checks/{run_id}/artefacts/{name}`), "Hard rules".
Rules: `.agents/skills/cqc-guideline/` "Private evidence".

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

## Acceptance criteria

- [ ] Returns `{ url, expires_at, content_type }`: a fresh presigned GET valid 15 minutes.
- [ ] `name` must appear in the run's own listing; anything else (including `../`, absolute or encoded traversal) → `404 artefact_not_archived`. Tests cover the traversal cases.
- [ ] A cited artefact missing from the bucket → `404 artefact_not_archived`.
- [ ] The bucket stays private; nothing returns a public or long-lived URL.

## Blocked by

- cqc-be-05-lo-history-and-run-report-e7a7ca
