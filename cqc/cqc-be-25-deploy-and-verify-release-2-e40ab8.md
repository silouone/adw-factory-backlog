---
id: cqc-be-25-deploy-and-verify-release-2-e40ab8
type: manual
status: queued
priority: 1
created: 2026-09-29
depends: [cqc-be-17-does-go1-provider-id-equal-the-asset-portal-9f2b04, cqc-be-18-observer-channels-make-fail-block-reachable-6ae7d9, cqc-be-19-the-runner-container-replaces-e2b-4d17bb, cqc-be-20-batch-queue-and-fargate-compute-8c4f2e, cqc-be-21-a-run-has-a-life-before-it-publishes-1b93a7, cqc-be-22-staff-trigger-a-check-and-a-batch-5e0c19, cqc-be-23-batch-progress-endpoint-0a76d4, cqc-be-24-batch-state-changes-drive-the-lifecycle-b2df76]
attempts: []
---
# Deploy release 2 and trigger a real check from the CMC

**Operator-executed.** Touches AWS accounts, ECR and a billed compute environment.

## Before the first trigger

- **B2 needs an owner and a date** (BE-7). `launch` mode is disabled in `/config` with a
  reason; until the analytics-excluded learner exists, a `launch` run can never Pass. Decide
  whether release 2 ships `preview` only.
- The **`go1-scormassets` grant** must be live, for the *correct* region — `ap-southeast-2`,
  see `cqc-be-15`. The runner needs it to fetch a package for an LO that has never run.
- `cqc-be-17`'s finding must be recorded as `BE-39`: if `core.provider_id` is not the asset
  portal, the never-run trigger path needs whatever that decision says instead.

## Steps

- Build and push the runner image to ECR; `sls deploy --stage dev`.
  **Measured 2026-09-30:** `content-quality-checker--ecr-runner-dev` is **empty**, and the
  deployed job definition (revision 1) pulls `:bootstrap`, the default of
  `${param:runnerImageTag, 'bootstrap'}`. Either push the image as `bootstrap`, or push an
  immutable tag and deploy with `--param runnerImageTag=<tag>`. Until then every job fails
  with `CannotPullContainerError`, which is not an egress problem (see `cqc-be-26`,
  rejected). Confirm with `aws ecr describe-images` before the first trigger.
- From the CMC: paste one LO id, press Check.
- Then paste a batch — include a deliberately bad id, an already-running id, and enough ids
  to cross `max_lo_ids_per_batch`.

## Done when

- [ ] A single check goes `queued → provisioning → running → succeeded` and the row appears
      in the CMC without a page reload (`cqc-fe-11`'s polling driving it).
- [ ] A batch returns `runs` and `rejected`, and every `reason` we can provoke is provoked.
- [ ] A run that fails in the container reaches `failed` or `preflight_failed` with a
      `status_reason` — kill one on purpose.
- [ ] At least one run reaches **`Fail-block`** on real evidence. If every run is still
      `Needs-review`, `cqc-be-18` did not achieve its purpose and release 2 is not done.
- [ ] `triggered_by` shows the operator's Go1 user id.
- [ ] Cost: the observed spend for the batch is within the BE-24 estimate. If not, say by
      how much — the cap values are SSM and changeable without a deploy.

## Blocked by

- all of cqc-be-17 .. cqc-be-24
