---
id: cqc-be-39-the-go1-asset-grant-covers-every-stage-704a18
type: manual
status: queued
priority: 1
created: 2026-09-30
depends: []
attempts: []
---
# Go1 grants read access on `go1-scormassets` to the four CQC roles, for every stage

**Operator-executed** (Silou → Go1 DevOps, Slack). Sources: BE-35, BE-36; `cqc-be-15`
(region), `cqc-be-27` (role list correction).

## The ask

- Bucket: `go1-scormassets`, Go1 account `769005983919`, **ap-southeast-2**. An earlier
  request may have said us-east-1.
- Coorp account: `446570804799`.

| role | actions | prefix |
|---|---|---|
| `content-quality-checker--role-runner-{stage}` | `s3:ListBucket` + `s3:GetObject` | `qa/*` (dev, staging), `production/*` (prod) |
| `…-role-projector-{stage}` | List + Get | same |
| `…-role-refresher-{stage}` | List + Get | same |
| `…-role-lifecycle-{stage}` | **`s3:ListBucket` only** | same |

- `api` does **not** need it.
- Staging and prod roles don't exist until those stacks deploy (`cqc-be-42`), and S3 rejects a
  policy naming a missing principal. Either grant dev now and the rest later, or use one
  statement with `StringLike aws:PrincipalArn` =
  `arn:aws:iam::446570804799:role/content-quality-checker--role-*`.
- Ask DevOps:
  - Is the bucket SSE-KMS with a customer-managed key? If so, the roles also need
    `kms:Decrypt`.
  - Is there a VPC-endpoint or SCP condition? The runner reaches S3 over the internet from
    Fargate.

## Done when

- [ ] Dev: `aws s3 ls s3://go1-scormassets/qa/` succeeds, run as the runner role.
- [ ] The refresher's next hourly run lists revisions without an access error.
- [ ] Staging and prod: granted, or explicitly deferred, with the date written here.

## Blocked by

- (nothing; the dev grant can go now)
