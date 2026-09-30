---
id: cqc-be-44-decide-where-backend-lives-b63a2c
type: manual
status: queued
priority: 3
created: 2026-09-30
depends: []
attempts: []
---
# Decide where `backend/` moves when it leaves the POC repo

**Decision ticket (operator).** Sources: BE-12 (`decisions-2026-09-26.md:82`), BE-17 (one
redaction gate).

## Coupling to cut

- `backend/src/common/contracts.ts:1` imports `poc0/observability/playability-report.mjs`.
- `contracts.ts:5` imports `scripts/context-extract/redaction.mjs`. BE-17 requires **one**
  redaction gate.
- The runner image copies `poc0`, `scripts/context-extract/*`, `scripts/e2b/*` and
  `.agents/skills/scorm-playability` (`scripts/runner/Dockerfile:28-34`).

## To decide

- The new home: a new repo, or a monorepo package.
- How the redaction module and report asserter are shared: a published package, or vendored
  with a drift test.
- Whether the runner moves with it.

## Done when

- [ ] The decision is recorded.
- [ ] An extraction ticket is written, with a red-first seam: a test asserting that no import
      escapes `backend/` (`../../../` in `src`).

## Blocked by

- (nothing)
