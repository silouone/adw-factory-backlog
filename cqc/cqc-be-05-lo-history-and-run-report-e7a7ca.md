---
id: cqc-be-05-lo-history-and-run-report-e7a7ca
type: feat
status: in-review
priority: 1
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-02-staff-only-access-authorizer-2b1d03, cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: [{"runId":"cqc-be-05-lo-history-and-run-report-e7a7ca-1790632342636","branch":"adw/cqc-be-05-lo-history-and-run-report-e7a7ca","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-05-lo-history-and-run-report-e7a7ca-1790632342636/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/17","provider":"codex","model":"gpt-5.6-sol"}]
---
# Staff open one LO's run history and one run's full report

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints" (`GET /cqc/content/{lo_id}/runs`, `GET /cqc/checks/{run_id}`);
BE-24a, BE-31b. Rules: `.agents/skills/cqc-guideline/` "Pass-through contracts".

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

- [ ] `GET /cqc/content/{lo_id}/runs` returns `{ items: RunSummary[] }` newest first (GSI `lo-runs`); an LO with no run → `404 lo_not_found`.
- [ ] `GET /cqc/checks/{run_id}` returns `{ run, metadata, report, artefacts }`; `metadata` and `report` are the S3 documents **byte-identical** (test compares bytes), `null` when absent.
- [ ] `artefacts` is `[{ name, content_type }]` from a listing of the run's prefixes.
- [ ] Unknown run → `404` with the shared error shape.

## Blocked by

- cqc-be-02-staff-only-access-authorizer-2b1d03
- cqc-be-03-published-run-becomes-a-row-projector-125f20
