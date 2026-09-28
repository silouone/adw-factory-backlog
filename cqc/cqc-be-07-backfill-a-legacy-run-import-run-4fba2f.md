---
id: cqc-be-07-backfill-a-legacy-run-import-run-4fba2f
type: feat
status: queued
priority: 1
created: 2026-09-28
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-00-redaction-scan-without-side-effects-4de841, cqc-be-03-published-run-becomes-a-row-projector-125f20]
attempts: []
---
# An engineer backfills a legacy run into a stage (import-run)

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Writers → import-run", "Field mapping" (`captured_at`, `origin`); BE-5,
BE-8, BE-16, BE-17. Rules: `.agents/skills/cqc-guideline/` "One gate into a bucket", "Stage isolation".

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
