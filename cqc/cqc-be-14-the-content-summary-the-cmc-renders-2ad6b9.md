---
id: cqc-be-14-the-content-summary-the-cmc-renders-2ad6b9
type: feat
status: in-progress
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 120, turns: 600, stallMinutes: 25}
depends: []
attempts: []
---
# `GET /cqc/content` returns the summary the CMC's backlog cards actually render

**This ticket carries a spec amendment (BE-38). Read it before building.** Sources:
`docs/backend/spec-cqc-backend-release-1.md` §`GET /cqc/content`; `backend/openapi.yaml`
`ContentListResponse.summary`; the CMC's `docs/cqc/backend-contract.md` line 117 and
`src/services/ContentQuality.service.ts` `ContentSummary`.

## The contradiction

The CMC crashes on first contact with the deployed API — React unmounts the whole tree:

```
Uncaught TypeError: Cannot read properties of undefined (reading 'Fail-block')
  at CheckedContentSummary   (CheckedContentSummary.tsx:16, reads summary.by_status["Fail-block"])
```

Two specs, both written 2026-09-26, independently, disagree on one field. Each side
implemented its own source faithfully:

| source | shape |
| --- | --- |
| `docs/cqc/backend-contract.md:117` — the CMC built this | `{los_checked, by_status:{Fail-block,Needs-review,Pass}, running, failed_to_run, open, resolved, revision_changed, partners_affected}` |
| `spec-cqc-backend-release-1.md` + `openapi.yaml` — this backend built this | `{total, "Fail-block", failed_to_run, running, "Needs-review", Pass}` |

A field-by-field diff of every live payload against the CMC's TypeScript interfaces was run
on 2026-09-29: `ContentRow`, `RunSummary`, `ContentRunsResponse`, `CheckDetailResponse` and
`ArtefactSummary` all match exactly; `ContentFacets`, `ContentQualityConfig`, `RunMetadata`
and `PlayabilityReport` differ only by extra fields the CMC ignores. **`summary` is the
single breaking divergence in the whole contract.**

## Proposed amendment — BE-38

> **BE-38 — the read API's `summary` is the CMC's backlog summary.** `GET /cqc/content`
> returns the shape published in the CMC's `docs/cqc/backend-contract.md`, superseding the
> `summary` definition in `spec-cqc-backend-release-1.md` §`GET /cqc/content`. Rationale:
> the CMC renders four backlog cards — Open, Fail-block, Revision changed, Partners
> affected — and the release-1 backend shape can populate exactly one of them.
> `backend/openapi.yaml` becomes the single contract for both repos; the CMC's
> `backend-contract.md` is retired once the CMC generates its types from it.

Record BE-38 in `docs/backend/decisions-2026-09-26.md`, update
`spec-cqc-backend-release-1.md` §`GET /cqc/content`, and update `backend/openapi.yaml`
**in the same PR as the code**. Do not silently diverge; if building shows BE-38 is wrong,
stop and say so in the final message.

## The shape to produce

Over the **unfiltered** row set, as today:

```
summary: {
  los_checked: number,                  // every LO with at least one run
  by_status: { "Fail-block": number, "Needs-review": number, Pass: number },
  running: number,
  failed_to_run: number,
  open: number,                         // rows whose case_state is not resolved
  resolved: number,                     // rows with an active resolution
  revision_changed: number,             // rows with revision_changed === true
  partners_affected: number | null      // distinct provider_id over open rows; null when unknowable
}
```

`open` + `resolved` and the `case_state` facet must agree: the CMC's tabs (All / Open /
Resolved) read the same numbers the rows carry.

## Acceptance criteria

- [ ] `GET /cqc/content` returns the shape above; `summary` still ignores the query filters.
- [ ] `partners_affected` is `null`, not `0`, when `provider_id` is unknown for every open
      row — the CMC prints `N/A` for null and a count for a number.
- [ ] `revision_changed` counts only `true`; `null` (unknown revision) is not counted.
- [ ] Unit tests cover: no rows; every status present; a row resolved then reopened;
      rows whose `provider_id` is null.
- [ ] `openapi.yaml` and the two spec files are updated in the same PR; `openapi:check` passes.

## Verify by hand after the deploy (operator)

The CMC's Content quality page renders its four backlog cards and its All/Open/Resolved
tab counts without a shim. Against the three runs imported on 2026-09-29 that is
Open 3 · Fail-block 1 · Revision changed 0 · Partners affected N/A, and tabs All 3 /
Open 3 / Resolved 0.

## Blocked by

- (nothing)
