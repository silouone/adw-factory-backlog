---
id: cqc-be-21-a-run-has-a-life-before-it-publishes-1b93a7
type: feat
status: in-progress
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: []
---
# A run exists in the read model before it finishes

Sources: BE-23 (the lifecycle enum), the CMC contract items 2 and 4.

## Why

BE-23: *"**Release 1 is projector-only**: the only writer is the projector, and it only sees
runs that finished and published, so every release-1 row is `succeeded` (or `failed`)."*

The CMC is already built for the other states. `cqc-fe-11` polls while a run is non-terminal,
`cqc-fe-10` disables the re-run action while an LO has a run in progress, and the row renders
`active_run`, `progress` and `triggered_by`. Today none of those can ever be non-null, so
that UI has never executed against real data.

## Scope

A second writer to the read model, alongside the projector:

- A **`queued` row** written the moment a check is accepted, before any container starts —
  so the LO's `active_run` is non-null immediately and the CMC's in-progress strip lights up.
- The full lifecycle from BE-23 — `queued | provisioning | preflight | running | finalising |
  succeeded | failed | preflight_failed | timeout | cancelled` — with `status_reason`
  **required on every failure state**.
- **`progress`** on non-terminal runs: `{advances, positions, cost_so_far_usd, updated_at}`.
- **`triggered_by`**: the Go1 user id the authorizer already resolves and puts in the request
  context. The read path must not call Go1 for it.

The projector keeps its job: it owns the terminal transition when a run publishes. Make sure
the two writers cannot fight — say in the PR which one wins and why.

## Acceptance criteria

- [ ] A `queued` row is visible through `GET /cqc/content` before any container runs, and the
      LO's `active_run` points at it.
- [ ] Every failure state carries a `status_reason`; a test asserts that, per state.
- [ ] `progress` is present on non-terminal runs and absent (not zeroed) on terminal ones.
- [ ] `triggered_by` comes from the authorizer context, never from a Go1 call on the read path.
- [ ] Projector idempotence is preserved: replaying a publish over a row that already
      transitioned does not corrupt it.
- [ ] `openapi.yaml` describes the lifecycle states and the new fields; `openapi:check` passes.

## Blocked by

- (nothing — this is read-model work and can run alongside the container tickets)
