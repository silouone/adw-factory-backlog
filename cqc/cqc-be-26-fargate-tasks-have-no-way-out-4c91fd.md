---
id: cqc-be-26-fargate-tasks-have-no-way-out-4c91fd
type: bug
status: rejected
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-be-26-fargate-tasks-have-no-way-out-4c91fd-1790785050847","branch":"adw/cqc-be-26-fargate-tasks-have-no-way-out-4c91fd","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-26-fargate-tasks-have-no-way-out-4c91fd-1790785050847/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# A Batch job cannot pull its own image: the Fargate tasks have no egress

> **REJECTED 2026-09-30: the premise is false.** The run (`…-1790785050847`) blocked at
> `red-check` because no honest red test exists, and the agent was right not to fake one:
> - `AssignPublicIp` is not a property of Batch `ComputeResources`. For Fargate it lives on the
>   job definition's `ContainerProperties.NetworkConfiguration`. This ticket read the compute
>   environment, so it saw "NOT SET".
> - Already there: `backend/serverless.yml` sets `AssignPublicIp: ENABLED` on
>   `RunnerJobDefinition` since cqc-be-20 (`b1bf523`), and `test/serverless-batch.test.ts`
>   ("Fargate job is x86 4 vCPU…") asserts it.
> - Deployed: `aws batch describe-job-definitions content-quality-checker--batch-job-dev` has
>   revision 1 with `assignPublicIp: ENABLED`.
>
> **The real pull risk is different:** the job definition pulls `…ecr-runner-dev:bootstrap`
> (`runnerImageTag` defaults to `bootstrap`), and the ECR repo is **empty**. See the note in
> `cqc-be-25`.

Measured against the **deployed** `dev` stack immediately after the release-2 deploy
(2026-09-30, `content-quality-checker-dev`, 108 resources). This blocks `cqc-be-25` — the
first real trigger will fail before the runner executes a line.

## The bug

`cqc-be-20` built a VPC with two **public** subnets and an internet gateway, which is the
right, cheap shape — no NAT, so no hourly charge. But the compute environment does not ask
for a public IP, and without one a task in a public subnet has no route out at all:

```
compute env : content-quality-checker--batch-compute-dev   VALID / ENABLED / FARGATE / maxvCpus 20
subnets     : subnet-02137315277c09dd9, subnet-035c8a4e522de8e09
route table : 10.80.0.0/16 -> local
              0.0.0.0/0    -> igw-019e2e6ba2965410a      (public subnet, no NAT)
VPC endpoints in vpc-0ea068531347a082d : 0
assignPublicIp : NOT SET  -> AWS defaults it to DISABLED
```

No public IP, no NAT, no VPC endpoints. A Fargate task therefore cannot reach **ECR** to
pull the runner image, nor S3, SSM, Bedrock or the Go1 gateway once it is running. Every
submitted job will fail at startup, most likely `CannotPullContainerError`.

## Why the gates missed it

`serverless package` renders the template and `config:check` validates it; neither can tell
that a Fargate task in a public subnet without `assignPublicIp` has no route out. It is only
visible against deployed resources, or by reading the three settings together.

## Red first (Art. I)

Assert it in the rendered configuration, where a test can see it: the Batch compute
environment's `ComputeResources` must declare `AssignPublicIp: ENABLED` **whenever** its
subnets are public and no NAT gateway or VPC endpoint exists. A test that simply asserts the
literal string is weaker but acceptable if it names, in its own text, the three facts that
make it necessary — a future reader must not "simplify" it away.

## Acceptance criteria

- [ ] `AssignPublicIp: ENABLED` on the compute environment's `computeResources`.
- [ ] A serverless-level test fails if it is removed, and its name or comment records why:
      public subnets + no NAT + no VPC endpoints.
- [ ] **Do not add a NAT gateway.** It would work, and it costs roughly $32/month per AZ for
      a dev stack that runs a handful of jobs. If you believe a NAT is required anyway, stop
      and say why rather than adding one.
- [ ] The security group still allows only what the runner needs outbound; this ticket does
      not widen ingress.
- [ ] `config:check` passes and `serverless package --stage dev` renders.

## Verify by hand after the deploy (operator)

Submit one Batch job with any image tag that exists in the ECR repository and watch it reach
`RUNNING` rather than failing at the pull.

## Blocked by

- (nothing)
