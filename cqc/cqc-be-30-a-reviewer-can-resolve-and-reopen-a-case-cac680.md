---
id: cqc-be-30-a-reviewer-can-resolve-and-reopen-a-case-cac680
type: feat
status: in-review
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-be-28-config-says-which-case-actions-exist-40185e, cqc-be-29-case-state-is-derived-from-run-history-0faae8]
attempts: [{"runId":"cqc-be-30-a-reviewer-can-resolve-and-reopen-a-case-cac680-1790802597181","branch":"adw/cqc-be-30-a-reviewer-can-resolve-and-reopen-a-case-cac680","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-30-a-reviewer-can-resolve-and-reopen-a-case-cac680-1790802597181/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/48","provider":"codex","model":"gpt-6-sol"}]
---
# A reviewer can resolve a case with an outcome and a note, and reopen it

Sources: FE D-26, D-16; CMC user stories 50-54 (`spec-cqc-fe-release-1.md:111-115`); CMC
`docs/cqc/backend-contract.md:119-120, 155`; BE-1, BE-14. **The CMC is already built against
this.** Read `src/services/ContentQuality.service.ts:49-72, 472-497` and
`common/CaseActions.tsx` on `cqc/release-1` before designing anything.

## Contract (fixed by the CMC)

- `POST /cqc/content/{lo}/resolution`
  - Body: `{ outcome, note }`.
  - `outcome` is one of `fixed_verified | validated_manually | false_positive | accepted_risk`.
  - Returns `202 Resolution`:
    `{ outcome, note, by, at, run_id, active, reopened_by_run_id }`.
  - `409 { error: "latest_run_not_pass" }` when `fixed_verified` is sent and the latest run
    is not Pass.
- `DELETE /cqc/content/{lo}/resolution` returns `200 { reopened: boolean }`.
- `ContentRow.resolution` carries the active or last resolution. Today it is pinned to `null`
  in these places, and each must change:
  - `types/runs.ts:119`
  - `decoders.ts:1103` (and the test `decoders.test.ts:1218`)
  - `run-transitions.ts:96`
  - `get-content/listing.ts:165`
  - `openapi.yaml:832-834`
- Errors use the existing `{ error, request_id }` body (`run-lifecycle/handler.ts:107`).

## Rules

- **Operator decision 2026-09-30:** only a newer, succeeded **Fail-block** run auto-reopens a
  manual resolution.
  - It sets `active: false`, `reopened_by_run_id`, and `case_state: open`, inside
    `publishTransition`, in `captured_at` order.
  - Needs-review, crashed runs and `revision_changed` do **not** reopen. The CMC shows banners
    for those.
- `by` = the authorizer context `user_id`, via `decodeAuthenticatedUserId`
  (`decoders.ts:676-686`).
- `at` = server time. `run_id` = the LO's latest run.

## Defaults (each defaulted 2026-09-30; the operator may override)

- A `POST` on an already-active resolution overwrites it and keeps the old one in history.
- A `DELETE` with nothing active returns `200 { reopened: false }`.
- `202` is kept, because the contract says so and the FE accepts any 2xx.
- `400 { error: "invalid_resolution" }` for:
  - a trimmed note shorter than 10 characters;
  - an unknown `outcome`.
- A resolution is allowed on a `none` LO: Validate shows for a Pass with `case_state: none`.
  It **requires a latest run**; with none, return `409 { error: "no_latest_run" }`.
- Store design:
  - An embedded `resolution` on the `LO#<id>`/`LO` item.
  - History items `LO#<id>`/`RES#<at>`.
  - History items must **not** carry `captured_at` or `_list`. Both GSIs project ALL and
    would list them (`serverless.yml:725-770`).

## The write-version trap

The projector's `TransactWriteCommand` rewrites every non-enrichment LO field under
`write_version = :expected` (`common/ports/transact-run-transition.ts:15-41`). A resolution
write that neither checks nor bumps `write_version` is silently clobbered by the next
projection. The resolution write must be conditional on `write_version`, bump it, and retry
once on a conflict.

## Red first (Art. I)

- `publishTransition` with an active resolution, then a newer Fail-block → reopened.
- Then a newer Needs-review → still active.
- A handler test with fake ports:
  - `409 latest_run_not_pass`;
  - `by` taken from the authorizer.
- A port test: a projector transact after a resolution write → the resolution survives.

## Acceptance criteria

- [ ] Both routes exist behind the staff authorizer, with `cors` declared (see `cqc-be-12`), and
      IAM on the table.
- [ ] Setting a resolution sets `case_state: resolved`. It moves to the Resolved tab and
      summary, and `partners_affected` drops it (`listing.ts:88`).
- [ ] Auto-reopen happens only on Fail-block, as above.
- [ ] History is kept. `GET /cqc/content` returns the active resolution, or the last one
      with `active: false`.
- [ ] `cqc-be-28`'s `actions.resolve` and `actions.reopen` flip to `enabled: true` in this PR.
- [ ] `openapi.yaml` covers both routes and the `Resolution` schema, with
      `scripts/validate-openapi.mjs` tables updated; `openapi:check` passes.
- [ ] `docs/backend/spec-cqc-backend-release-1.md:181, 186` are updated.

## Blocked by

- cqc-be-28-config-says-which-case-actions-exist-40185e
- cqc-be-29-case-state-is-derived-from-run-history-0faae8
