---
id: cqc-be-31-limitations-say-when-they-are-shared-2fb535
type: feat
status: in-review
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: [{"runId":"cqc-be-31-limitations-say-when-they-are-shared-2fb535-1790802610549","branch":"adw/cqc-be-31-limitations-say-when-they-are-shared-2fb535","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-31-limitations-say-when-they-are-shared-2fb535-1790802610549/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/46","provider":"codex","model":"gpt-6-sol","rebased":"84089470c9e40fb22691d123e49a149b9aedb1a9"}]
---
# Each limitation says whether it is shared by every run of its profile

Sources: CMC spec "Shared limitations" (`spec-cqc-fe-release-1.md:262-266`),
`backend-contract.md:158`, `open-questions.md:32`; CMC D-5 (the report is returned unchanged).
**Operator decision 2026-09-30:** use a **static catalogue** per `profile_id`, not the
prototype's ≥60% statistic.

## Shape

`GET /cqc/checks/{run_id}` gains a top-level `limitations_meta: [{ text, shared }]`, in the
same order as `report.limitations`.

**`report` stays byte-identical** (D-5, `get-check/response.ts:12-18`). Don't mutate
`report.limitations`; the CMC renders it as `string[]`.

## Catalogue

- It is a pure `classifyLimitations(profileId, limitations) → {text, shared}[]`, with the
  catalogue as data beside it.
- Seed it from the structural strings the Go1 mapper emits for profile `go1-interactive-li`
  (`poc0/observability/playability-from-nav-run.mjs:257-279, 304`):
  - empty package snapshot
  - null `content_hash`
  - desktop only
  - `fail_path` untested
  - preview mode (B2)
- Variable strings, such as the declared stop reason with `LESSONS_DONE=`, match on a stable
  prefix or are not shared.
- An unknown profile means everything is `shared: false`.
- Import the strings from the mapper, or pin them in a test that fails when the mapper's text
  changes. The catalogue must not drift silently.

## Red first (Art. I)

- A table test for `classifyLimitations`.
- A `checkResponseBody` test: `report` has an equal sha, and `limitations_meta` is present
  with one entry per limitation.

## Acceptance criteria

- [ ] `limitations_meta` appears in the response and in `openapi.yaml`
      (`CheckResponse` ~:987); `openapi:check` passes.
- [ ] The catalogue is keyed by `provenance.profile_id` from the report.
- [ ] A drift test ties the catalogue to the mapper's strings.

## Blocked by

- (nothing)
