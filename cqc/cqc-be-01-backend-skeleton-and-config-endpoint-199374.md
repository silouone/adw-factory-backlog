---
id: cqc-be-01-backend-skeleton-and-config-endpoint-199374
type: feat
status: done
priority: 1
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-00-redaction-scan-without-side-effects-4de841]
attempts: [{"runId":"cqc-be-01-backend-skeleton-and-config-endpoint-199374-1790604997436","branch":"adw/cqc-be-01-backend-skeleton-and-config-endpoint-199374","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-01-backend-skeleton-and-config-endpoint-199374-1790604997436/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"},{"runId":"cqc-be-01-backend-skeleton-and-config-endpoint-199374-1790609073950","branch":"adw/cqc-be-01-backend-skeleton-and-config-endpoint-199374","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-01-backend-skeleton-and-config-endpoint-199374-1790609073950/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/14","provider":"codex","model":"gpt-5.6-sol"}]
---
# The backend serves GET /cqc/config (tracer bullet on the scaffold)

**The `backend/` scaffold (PR #13, cqc-be-01a) is merged on `main`** and the factory's setup runs
`npm --prefix backend ci`, so `node_modules` is installed in your workspace. That PR holds every
release-1 dependency and the lockfile, because the agent sandbox has no network. This ticket only
writes code and tests on top of it. (Attempt 1 stopped correctly because #13 was not merged yet.)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Resources per stage", "Endpoints"
(`GET /cqc/config`); BE-7, BE-9, BE-13, BE-18, BE-24, BE-25; rules in `.agents/skills/cqc-guideline/`
(read `SKILL.md` and `references/typescript.md` first).

## Already in `backend/` (from #13)

- `package.json` + lockfile: io-ts, fp-ts, AWS SDK v3 (DynamoDB, lib-dynamodb, S3, s3-request-presigner,
  SSM), Vitest + coverage, Biome, Serverless 3 + serverless-esbuild, aws-sdk-client-mock, tsx.
  **Do not add, remove or upgrade any dependency**, and do not run `npm install`: it cannot work offline.
- Scripts: `typecheck`, `lint`, `test` (coverage threshold **100%**), `config:check`, `package:dev`.
- `serverless.yml` stub with `custom.resourceNames`, `go1ApiUrl`, `scormAssetsPrefix`, ESM esbuild,
  and `functions: {}`.
- `src/common/resource-names.ts`: `STAGES`, `Stage`, `isStage`, `resourceName(type, name, stage)`.

## What to build

The first real lambda, end to end, in the layout of `references/typescript.md`:

- `src/common/errors.ts`: named error classes; messages name the operation and the ids touched.
- `src/common/utils/logger.ts`: structured JSON (`service: "cqc"`, `lambda`, `stage`, `request_id`); never logs secrets.
- `src/common/decoders.ts`: io-ts codecs, starting with the SSM `config` document.
- The shared HTTP error shape `{ error, message?, request_id }` (`request_id` = API Gateway request id).
- `src/get-config/`: a port reading SSM `/credentials/{stage}/content-quality-checker/config`
  through `createPorts`, the decoder, and a handler returning the spec's shape (default portal
  `36743531`, `launch` disabled with its `disabled_reason`, agents, estimate, limits,
  `verdict_scope: "navigation-only"`).
- `serverless.yml`: the `getConfig` function on `GET /cqc/config` (HTTP API), role and SSM read scoped to
  this stage's prefix. The authorizer comes in cqc-be-02; leave a clear hook for it.

## Acceptance criteria

- [ ] `GET /cqc/config` returns the spec's shape from a decoded SSM document (handler test with a faked port).
- [ ] An undecodable SSM document fails with a named error naming the bad key, mapped to a `500` in the shared error shape with `request_id`.
- [ ] A missing SSM parameter fails with a named error naming the parameter path.
- [ ] Every named error has a test proving it is thrown.
- [ ] `npm --prefix backend run package:dev` succeeds (serverless-esbuild now has a function to bundle); add it to the scripts the gates can run if needed.
- [ ] Coverage stays at 100%; no secret in `serverless.yml`, env files, logs or fixtures.

## Verify

The factory gates run the two repo selftests and, in `backend/`: `typecheck`, `lint`, `test`
(100% coverage) and `config:check`.

## Blocked by

- cqc-be-00-redaction-scan-without-side-effects-4de841 (done)
- PR #13 merged (done)
