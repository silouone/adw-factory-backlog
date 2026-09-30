---
id: cqc-be-20-batch-queue-and-fargate-compute-8c4f2e
type: feat
status: in-review
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [cqc-be-19-the-runner-container-replaces-e2b-4d17bb]
attempts: [{"runId":"cqc-be-20-batch-queue-and-fargate-compute-8c4f2e-1790755548814","branch":"adw/cqc-be-20-batch-queue-and-fargate-compute-8c4f2e","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-20-batch-queue-and-fargate-compute-8c4f2e-1790755548814/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/34","provider":"codex","model":"gpt-6-sol"}]
---
# The infrastructure that runs a container: Batch queue, Fargate compute, runner role

Sources: BE-19, BE-24 (`concurrency: 5`, `AGENT_TIMEOUT_S: 2700`), BE-27 (per-stage roles),
BE-36 (role names were fixed before the first deploy).

Release 1 deliberately created **no** compute: no ECS, no Batch, no Fargate. This ticket adds
the first of it.

## Scope, in `backend/serverless.yml` with the rest of the stack

- **ECR repository** for the `cqc-be-19` image, per stage or shared — say which and why.
- **AWS Batch job queue** + **Fargate compute environment**, x86, **4 vCPU / 8 GB** per job.
- **Job definition** parameterised by `RUN_ID` and `LO_ID`, with the per-run timeout from
  BE-24 (`AGENT_TIMEOUT_S: 2700`, so 45 minutes) and `concurrency: 5` reflected in the
  queue's limits.
- **`content-quality-checker--role-runner-{stage}`** — the fourth role, whose name was fixed
  in BE-36 precisely so the Go1 cross-account grant could be requested up front. Scoped to
  that stage's bucket, table and SSM path, **plus Bedrock invoke in `eu-central-1`** (BE-27),
  **plus read on `go1-scormassets`** — note that bucket is in `ap-southeast-2`, see
  `cqc-be-15`.

## Acceptance criteria

- [ ] `serverless package --stage dev` renders the queue, compute environment, job
      definition, ECR repository and runner role.
- [ ] The runner role's name is exactly `content-quality-checker--role-runner-{stage}`
      (BE-36) — the grant request already names it.
- [ ] No role gets a resource wildcard; the three existing roles keep their current scope.
- [ ] A serverless-level test asserts the job definition's vCPU, memory, timeout and the
      queue's concurrency against the BE-24 values, so a change to the cost guard cannot
      silently drift from the infrastructure.
- [ ] `config:check` passes.

## Verify by hand after the deploy (operator)

Submit one job by hand with a known `RUN_ID` / `LO_ID` and watch it reach `SUCCEEDED`.

## Blocked by

- cqc-be-19-the-runner-container-replaces-e2b-4d17bb
