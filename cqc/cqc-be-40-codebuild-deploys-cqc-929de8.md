---
id: cqc-be-40-codebuild-deploys-cqc-929de8
type: manual
status: queued
priority: 3
created: 2026-09-30
depends: [cqc-be-38-a-deploy-is-one-command-01d28b, cqc-be-44-decide-where-backend-lives-b63a2c]
attempts: []
---
# CodeBuild deploys CQC from `main`, through `coorpacademy-infra`

**Operator + DevOps.** Sources: BE-18. Laptop deploys stay allowed for V1, so this does not
gate prod.

## Steps

- Add a `content-quality-checker` entry to `coorpacademy-infra`
  `config/serverless/serverless-definitions.yml`. It is modelled on `ai-translation` (:14-23):
  - `githubRepo`, `mainBranch: main`, `serverlessFolder: backend`
  - `deployCommand` per environment → `npm run deploy:{dev,staging,prod}` (`cqc-be-38`)
  - `extraEnvironments: [development]` mapped to `dev`
- The shared template (`codepipeline-serverless-deploy.cfn.yml`) runs `standard:7.0` with
  `PrivilegedMode: true` (:169-171), so Docker builds work.
- Confirm with DevOps:
  - The CodeBuild service role (from SSM `/cluster/codebuild/staging/serverless-service-role/name`)
    can create named IAM roles, Batch, ECR and API Gateway resources, and push to eu-west-1 ECR.
  - Whether prod is gated by manual approval (`ECSCodepipelineApprovalTopic`).
- A repo admin adds the GitHub webhook, secret `ssm:/credentials/global/jw/secret-key`
  (definitions :5-12).
- Settle Serverless v3 support and licence (`todo.md:29`).

## Done when

- [ ] A merge to `main` deploys dev with a commit-tagged runner image.
- [ ] Staging and prod deploys are defined, even if gated.

## Blocked by

- cqc-be-38-a-deploy-is-one-command-01d28b
- cqc-be-44-decide-where-backend-lives-b63a2c (don't build the pipeline twice)
