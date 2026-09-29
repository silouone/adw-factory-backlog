---
id: cqc-be-11-deploy-dev-and-verify-release-1-d3bcb0
type: manual
status: queued
priority: 1
created: 2026-09-28
depends: [cqc-be-02-staff-only-access-authorizer-2b1d03, cqc-be-04-list-checked-los-get-content-961b88, cqc-be-05-lo-history-and-run-report-e7a7ca, cqc-be-06-open-an-artefact-presigned-url-5ab9db, cqc-be-07-backfill-a-legacy-run-import-run-4fba2f, cqc-be-08-lo-exists-in-go1-enrichment-a0377d, cqc-be-09-revision-drift-and-connected-refresher-7c8fc8, cqc-be-12-cors-preflight-on-every-cqc-route-7b41e2, cqc-be-13-artefact-names-are-paths-so-none-can-be-opened-c93f08, cqc-be-14-the-content-summary-the-cmc-renders-2ad6b9]
attempts: []
---
# Deploy dev and run the release-1 verification

**Operator-executed.** Touches AWS accounts and secrets. Sources: `docs/backend/spec-cqc-backend-release-1.md` "Verification";
BE-20, BE-26, BE-27; `docs/backend/todo.md` "Owned by Silou" and "Facts to confirm before prod".

## Steps

- Copy the Go1 credentials into `/credentials/dev/content-quality-checker/go1-*` as SecureString (BE-20), and write the SSM `config` and `staff-allow-list`.
- `sls deploy --stage dev` with the `coorp` profile.
- `import-run --stage dev` three legacy runs, including one without a report.

## Progress, 2026-09-29

The stack is deployed and the credentials are in place. Findings and evidence:
`~/personal_project/adw-factory/ai_docs/2026-09-29-cqc-dev-integration-findings.md`.

Done:
- SSM `/credentials/dev/content-quality-checker/`: `go1-{root-jwt,client-cert,client-key,ca-cert}`
  copied byte-identical from `/credentials/dev/translated-language-system/` (SHA-256 matched),
  `config` (round-trips through `CqcConfigCodec`), `staff-allow-list` = `[]` per BE-14.
- `serverless deploy --stage dev` → `content-quality-checker-dev`,
  `https://jrgh8xo9ig.execute-api.eu-west-1.amazonaws.com/dev`, 8 lambdas, 5 endpoints.
- Three runs imported: `10622213/local-nav-…-aff88c`, `10672378/agent-nav-go1-…-a90613f1`,
  `10910102/local-nav-…-40c91f`. Redaction re-scan passed; 3 × `RUN#` + 3 × `LO#` in DynamoDB.
- A staff bearer for dev comes from `GET https://staff.qa.go1.cloud/api/user/account/current`
  with the `_oauth2_proxy_staff` cookie; the response's `jwt` field is the token.

Verification status: 1 ✅ · 2 negative cases ✅ (`401 invalid_token`, no token and bad token)
· 2 positive case ✅ at the API, ❌ in a browser (cqc-be-12) · 3 ✅ by curl, ❌ in a browser
(cqc-be-12) · 4 negative ✅, positive ❌ (cqc-be-13) · 5 not constructed yet · 6 still waiting
on the `go1-scormassets` grant.

Remaining operator work once 12–14 are merged and redeployed: re-run 2–5 from the CMC, and
import a fourth run with its report removed to exercise `failed` / `report_missing` (5).

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
