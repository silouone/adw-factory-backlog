---
id: cqc-be-22-staff-trigger-a-check-and-a-batch-5e0c19
type: feat
status: in-progress
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-be-20-batch-queue-and-fargate-compute-8c4f2e, cqc-be-21-a-run-has-a-life-before-it-publishes-1b93a7]
attempts: []
---
# `POST /cqc/checks` and `POST /cqc/batches`: the two buttons the CMC already has

Sources: the CMC contract items 6 and 7; BE-23, BE-24. **The frontend for this shipped in
`cqc-fe-10`.** Its paste bar and Check button are live on `cqc/release-1` today and call
these two paths; both currently return a raw API Gateway `403` because no route exists.
`src/services/ContentQuality.service.ts` `triggerCheck` / `triggerBatch` is the client —
read it, and match it, before designing the response.

## `POST /cqc/checks` — one LO

Body `{ lo_id, portal_id, launch_mode, agent }` → `202 { run_id, lo_id, status: "queued" }`.
`409 already_running` if the LO has an active run.

## `POST /cqc/batches` — many LOs pasted from Slack

Body `{ lo_ids[], portal_id, launch_mode, agent }` →
`202 { batch_id, runs: [{run_id, lo_id, status}], rejected: [{lo_id, reason}] }`.
`reason` is one of `not_a_numeric_lo_id | already_running | not_found_in_go1 | over_limit`.

## Order of operations (BE-23)

Write the `queued` row **first**, then `SubmitJob` with `RUN_ID`. Never submit a job that has
no row — a container running against a row that does not exist is unobservable.

## The cost cap is enforced here (BE-24)

`max_lo_ids_per_batch: 20`, `max_batch_cost_usd: 50`, `concurrency: 5` — read from SSM
`config`, not hard-coded, so it changes without a deploy. Over the cap → the offending ids
come back in `rejected` with `over_limit`; the rest still run. A batch is never rejected
whole because one id is bad.

## Acceptance criteria

- [ ] Both routes exist, are behind the staff authorizer, and **declare `cors`** with the
      stage's origins — see `cqc-be-12`; do not repeat that bug on a new route.
- [ ] `202` shapes match the contract exactly, because the CMC is already built against them.
- [ ] Every `rejected.reason` value is reachable and tested, including `not_found_in_go1`.
- [ ] A rejected id never gets a `queued` row and never gets a job submitted.
- [ ] `409 already_running` for a single check; for a batch the same LO lands in `rejected`.
- [ ] Cost-cap values come from SSM at request time.
- [ ] `openapi.yaml` covers both; `openapi:check` passes.

## Blocked by

- cqc-be-20-batch-queue-and-fargate-compute-8c4f2e
- cqc-be-21-a-run-has-a-life-before-it-publishes-1b93a7
