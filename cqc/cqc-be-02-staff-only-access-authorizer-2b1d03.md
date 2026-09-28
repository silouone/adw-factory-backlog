---
id: cqc-be-02-staff-only-access-authorizer-2b1d03
type: feat
status: done
priority: 1
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-01-backend-skeleton-and-config-endpoint-199374]
attempts: [{"runId":"cqc-be-02-staff-only-access-authorizer-2b1d03-1790611568176","branch":"adw/cqc-be-02-staff-only-access-authorizer-2b1d03","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-02-staff-only-access-authorizer-2b1d03-1790611568176/workspace","outcome":"in-review","provider":"codex","model":"gpt-5.6-sol","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/16"}]
---
# Only Go1 staff can call the CQC API

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Auth"; BE-14, BE-15. Rules: `.agents/skills/cqc-guideline/`.

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

A REQUEST Lambda authorizer on every `/cqc` route. It reads `Authorization: Bearer <jwt>` and
calls Go1 `GET {GO1_API_URL}/user/account/current` with it (Go1 prod for `prod`, Go1 QA for
`dev` and `staging`). Handlers receive `{ user_id }` from the Go1 response `id`, and every
request logs it.

## Acceptance criteria

- [ ] Missing header, or Go1 answering non-2xx → `401 { error: "invalid_token" }`.
- [ ] Roles without `"Admin on #Accounts"` → `403 { error: "not_staff" }`.
- [ ] Non-empty SSM allow-list that does not contain the user id → `403 not_staff`; an empty allow-list lets every staff user through.
- [ ] The result is cached for 5 minutes, keyed by `sha256(token)`.
- [ ] The token is never logged, stored or put in an error message (a test asserts this on the log output).
- [ ] `GET /cqc/config` is behind the authorizer, and any route added later is too by default.
- [ ] Vitest covers every outcome above with a faked Go1 port.

## Blocked by

- cqc-be-01-backend-skeleton-and-config-endpoint-199374
