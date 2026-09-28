---
id: cqc-be-01-backend-skeleton-and-config-endpoint-199374
type: manual
status: queued
priority: 1
created: 2026-09-28
depends: []
attempts: []
---
# The backend exists and serves GET /cqc/config (tracer bullet)

**Operator-built, in session (not dispatched):** this ticket creates the `backend/` scripts
that become the factory target's gates. The factory cannot run a ticket whose gates do not
exist yet. Built tests-first like any other ticket.

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Resources per stage", "Endpoints" (`GET /cqc/config`); BE-7, BE-9, BE-12,
BE-13, BE-18, BE-24, BE-25, BE-36; layout and rules in `.agents/skills/cqc-guideline/`.

## What to build

- `backend/` with Serverless Framework v3, TypeScript strict, esbuild, Vitest and Biome, and
  the scripts `typecheck`, `lint`, `test`. Node 22.
- Stages `dev`, `staging`, `prod` with the fixed TLS-style names
  (`content-quality-checker--<type>-<name>-<stage>`), including the four role names (BE-36).
- The shared error shape `{ error, message?, request_id }` and a structured JSON logger.
- `GET /cqc/config`: read the SSM `config` document through a port, decode it, and return the
  spec's shape (default portal `36743531`, `launch` disabled with its `disabled_reason`,
  limits, `verdict_scope: "navigation-only"`).

## Acceptance criteria

- [ ] `typecheck`, `lint` and `test` scripts exist and pass.
- [ ] `serverless package --stage dev` succeeds.
- [ ] `GET /cqc/config` returns the spec's shape from a decoded SSM document; an invalid document fails with a named error that names the bad key.
- [ ] No secret in `serverless.yml`, env files or fixtures.
- [ ] `targets/content-quality-checker.json` in adw-factory gains the backend gates.

## Blocked by

None. This ticket can start immediately.
