---
id: cqc-be-11-deploy-dev-and-verify-release-1-d3bcb0
type: manual
status: queued
priority: 1
created: 2026-09-28
depends: [cqc-be-02-staff-only-access-authorizer-2b1d03, cqc-be-04-list-checked-los-get-content-961b88, cqc-be-05-lo-history-and-run-report-e7a7ca, cqc-be-06-open-an-artefact-presigned-url-5ab9db, cqc-be-07-backfill-a-legacy-run-import-run-4fba2f, cqc-be-08-lo-exists-in-go1-enrichment-a0377d, cqc-be-09-revision-drift-and-connected-refresher-7c8fc8]
attempts: []
---
# Deploy dev and run the release-1 verification

**Operator-executed.** Touches AWS accounts and secrets. Sources: `docs/backend/spec-cqc-backend-release-1.md` "Verification";
BE-20, BE-26, BE-27; `docs/backend/todo.md` "Owned by Silou" and "Facts to confirm before prod".

## Steps

- Copy the Go1 credentials into `/credentials/dev/content-quality-checker/go1-*` as SecureString (BE-20), and write the SSM `config` and `staff-allow-list`.
- `sls deploy --stage dev` with the `coorp` profile.
- `import-run --stage dev` three legacy runs, including one without a report.

## Done when (spec "Verification" 1–5, and 6 once the DevOps grant exists)

- [ ] The three runs land, the redaction re-scan passes, `RUN#` and `LO#` are written.
- [ ] From the CMC on `staff-new.qa.go1.cloud`: staff JWT lists them in FE order; non-staff → `403 not_staff`; no token → `401 invalid_token`.
- [ ] `GET /cqc/checks/{run_id}` is byte-identical to S3.
- [ ] An artefact URL opens; a cited-but-missing one → `404 artefact_not_archived`.
- [ ] The run without a report shows `failed` / `report_missing`.
- [ ] (After the grant) `current_revision` and `connected` are filled for an imported LO.

## Blocked by

- cqc-be-02-staff-only-access-authorizer-2b1d03
- cqc-be-04-list-checked-los-get-content-961b88
- cqc-be-05-lo-history-and-run-report-e7a7ca
- cqc-be-06-open-an-artefact-presigned-url-5ab9db
- cqc-be-07-backfill-a-legacy-run-import-run-4fba2f
- cqc-be-08-lo-exists-in-go1-enrichment-a0377d
- cqc-be-09-revision-drift-and-connected-refresher-7c8fc8
