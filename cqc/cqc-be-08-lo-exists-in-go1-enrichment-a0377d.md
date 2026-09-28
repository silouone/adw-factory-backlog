---
id: cqc-be-08-lo-exists-in-go1-enrichment-a0377d
type: feat
status: queued
priority: 2
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: []
---
# Each LO row knows whether the LO exists in Go1 and who provides it

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints" (last paragraph), "Hard rules"; BE-11, BE-20, BE-22. Rules:
`.agents/skills/cqc-guideline/` "Read-only Go1", "Secrets from SSM at runtime".

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

After writing a run, the projector calls the Go1 content API
(`learning-objects/{lo}?include[]=core`, root JWT and mTLS credentials from SSM, reusing the
TLS client logic) and caches existence and `provider_id` on `LO#`. The read path never calls Go1.

## Acceptance criteria

- [ ] Found → `provider_id` stored; Go1 404 → the LO is marked not found; other failures leave the fields unchanged and log `{ lo_id, reason }`.
- [ ] Server-certificate verification is off on the Go1 gateway agent **only**, with a security TODO (BE-22); a test proves no other agent disables it.
- [ ] Only GET requests are made with the root JWT.
- [ ] Credentials are read from SSM at runtime; none appear in code, config, logs or fixtures.

## Blocked by

- cqc-be-03-published-run-becomes-a-row-projector-125f20
