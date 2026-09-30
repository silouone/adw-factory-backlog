---
id: cqc-be-23-batch-progress-endpoint-0a76d4
type: feat
status: in-progress
priority: 2
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 150, turns: 600, stallMinutes: 25}
depends: [cqc-be-22-staff-trigger-a-check-and-a-batch-5e0c19]
attempts: [{"runId":"cqc-be-23-batch-progress-endpoint-0a76d4-1790770824510","branch":"adw/cqc-be-23-batch-progress-endpoint-0a76d4","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-23-batch-progress-endpoint-0a76d4-1790770824510/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/36","provider":"codex","model":"gpt-6-sol"}]
---
# `GET /cqc/batches/{batch_id}`: what happened to the twenty ids I pasted

Source: the CMC contract item 8.

`{ batch_id, created_at, run_ids, counts: {<lifecycle>: n}, runs: RunSummary[] }`.

`counts` is keyed by the BE-23 lifecycle enum, so a caller can render "3 running, 1 failed,
16 queued" without walking `runs`.

## Acceptance criteria

- [ ] Behind the staff authorizer, **with `cors` declared** (see `cqc-be-12`).
- [ ] `404` with the service's own error shape for an unknown `batch_id` — not a raw
      gateway error.
- [ ] `counts` covers every lifecycle state present in the batch and omits the rest.
- [ ] `runs` reuses the same `RunSummary` shape as every other endpoint; no third variant.
- [ ] `openapi.yaml` covers it; `openapi:check` passes.

## Blocked by

- cqc-be-22-staff-trigger-a-check-and-a-batch-5e0c19
