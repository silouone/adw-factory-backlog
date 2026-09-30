---
id: cqc-be-24-batch-state-changes-drive-the-lifecycle-b2df76
type: feat
status: done
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [cqc-be-22-staff-trigger-a-check-and-a-batch-5e0c19]
attempts: [{"runId":"cqc-be-24-batch-state-changes-drive-the-lifecycle-b2df76-1790770814886","branch":"adw/cqc-be-24-batch-state-changes-drive-the-lifecycle-b2df76","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-24-batch-state-changes-drive-the-lifecycle-b2df76-1790770814886/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/37","provider":"codex","model":"gpt-6-sol"}]
---
# A run that dies in the container still tells the truth in the read model

Sources: BE-23; `cqc-be-19`'s exit-code contract.

## Why

Without this, a container that crashes leaves its row `queued` or `running` forever. The CMC
polls a non-terminal run indefinitely and content ops see a check that never finishes. The
publish path only covers the happy case — a run that publishes. This covers the rest.

## Scope

An EventBridge rule on **AWS Batch job state change**, targeting a lambda that maps the job's
state onto the run's lifecycle:

- `RUNNABLE` / `STARTING` → `provisioning`
- `RUNNING` → `running`
- `FAILED` with container exit **2** → `preflight_failed`
- `FAILED` with container exit **3** → `failed`
- `FAILED` on the Batch timeout → `timeout`
- `SUCCEEDED` → leave the terminal transition to the projector; the publish is the truth.

Every failure state carries a `status_reason` (BE-23) naming what the job reported.

## Acceptance criteria

- [ ] A crashed container's row reaches a terminal state without any publish.
- [ ] Exit 2 and exit 3 map to different states and different `status_reason` values.
- [ ] A Batch timeout is distinguishable from a container failure.
- [ ] `SUCCEEDED` does not race the projector into a wrong terminal state — say in the PR
      how the two are ordered.
- [ ] Idempotent: the same state-change event delivered twice does not corrupt the row.
- [ ] A serverless-level test asserts the rule's event pattern, so it cannot silently stop
      matching.

## Blocked by

- cqc-be-22-staff-trigger-a-check-and-a-batch-5e0c19
