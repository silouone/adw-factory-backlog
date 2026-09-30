---
id: cqc-be-42-staging-and-prod-are-deployed-ce77d9
type: manual
status: queued
priority: 2
created: 2026-09-30
depends: [cqc-be-25-deploy-and-verify-release-2-e40ab8, cqc-be-27-the-trigger-reads-the-go1-bucket-in-its-region-51db27, cqc-be-37-the-go1-gateway-certificate-is-verified-24925a, cqc-be-39-the-go1-asset-grant-covers-every-stage-704a18, cqc-be-41-cqc-has-its-own-go1-identity-9eddef]
attempts: []
---
# Staging and prod stacks exist and pass the release verification

**Operator-executed.** Only `dev` exists today. No release assigns staging or prod; BE-18
allows laptop deploys.

## Before the deploy (`todo.md` "Facts to confirm before prod")

- The prod CORS host `staff-new.go1.co` is inferred (`todo.md:26`, `serverless.yml:58-59`).
- The SSM path is `/credentials/prod/…` vs `/credentials/production/…`; both exist for TLS
  (`todo.md:27`). CQC uses `prod`.
- The portal token for `36743531` (`todo.md:19`).
- Fargate On-Demand quota: 20 vCPU per stage.
- Headless Chrome on Fargate is unverified (`scripts/runner/README.md:52`), so run the one-job
  smoke per stage.

## Steps

- Write SSM `config` and `staff-allow-list` per stage.
- Push the runner image to each stage's own ECR repo, then deploy with `runnerImageTag`
  (`cqc-be-38` scripts, when merged).
- Re-run the `cqc-be-11` and `cqc-be-25` verification lists against staging, then prod.

## Done when

- [ ] Staging is verified.
- [ ] Prod is verified.
- [ ] The URLs are recorded here and in the CMC's `APP_CQC_API_URL` per environment.

## Blocked by

- the depends list above
