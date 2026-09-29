---
id: cqc-be-13-artefact-names-are-paths-so-none-can-be-opened-c93f08
type: bug
status: queued
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# Every artefact name contains a slash, so no artefact can ever be opened

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints"
(`GET /cqc/checks/{run_id}/artefacts/{name}`); ticket `cqc-be-06`.
Measured live on `dev` on 2026-09-29 against run
`local-nav-10622213-20260923-124809-aff88c` (LO `10622213`).

## The bug

`artefactsFromKeys` (`src/common/artefacts.ts:29`) builds every name as
`` `${layer}/${key.slice(prefix.length)}` ``, so **every** name the listing publishes carries
its layer prefix — and the snapshot ones carry two segments:

```
output/raw-context.json      runs/agent-stream.jsonl      runs/playability-report.json
runs/snapshots/manifest.json runs/snapshots/step-0001.yml   ... 66 of 66 contain "/"
```

`GET /cqc/checks/{run_id}` returns all 66. But the fetch route's `{name}` is a **single,
non-greedy** API Gateway path parameter, and a REST API does **not** decode `%2F` before
handing the parameter to the lambda. `get-artefact` then does an exact
`artefacts.find((c) => c.name === name)` (`src/get-artefact/handler.ts`), which can never
match a listed name.

Measured — every form fails, and CloudWatch shows all four reaching the `.find()` step
(`lo_id` is present in each log line, so `getRun` and `listRunArtefacts` both succeeded):

```
.../artefacts/output%2Fraw-context.json    -> 404 artefact_not_archived
.../artefacts/output%252Fraw-context.json  -> 404 artefact_not_archived
.../artefacts/output%2fraw-context.json    -> 404 artefact_not_archived
.../artefacts/runs/agent-stream.jsonl      -> 403 (gateway: no such resource)
.../artefacts/does-not-exist.png           -> 404 artefact_not_archived   <- correct
```

So `cqc-be-06`'s negative half is right and its positive half has never worked. The
Evidence tab (`cqc-fe-08`) has nothing it can open, and spec Verification 4's positive case
cannot pass.

## Why the gates missed it

`cqc-be-06`'s unit tests feed the handler a `name` directly, so they exercise the comparison
with names the test itself chose. Nothing in the suite asserts that a name **taken from the
listing** survives a round trip through an API Gateway path parameter.

## Red first (Art. I)

The failing test is a round trip, not a new assertion on the old shape: take the artefacts
`listRunArtefacts` produces for a realistic key set (including a `runs/snapshots/…` two-segment
name), put each name through whatever encoding the route requires, and assert `get-artefact`
resolves it to the right S3 key. It must go red for every prefixed name today.

## Acceptance criteria

- [ ] Every name returned by `GET /cqc/checks/{run_id}` can be fetched by
      `GET /cqc/checks/{run_id}/artefacts/{name}`, including two-segment
      `runs/snapshots/…` names.
- [ ] `cqc-be-06`'s guarantees are unchanged: a name absent from the run's own listing —
      including `../`, absolute and encoded traversal — still returns
      `404 artefact_not_archived`, and the bucket stays private behind a 15-minute presign.
- [ ] Whatever the chosen route/encoding, the published contract in `backend/openapi.yaml`
      describes it, and `openapi:check` passes.

Likely fix, not prescribed: decode the path parameter once before matching and keep the
listing-membership check as the traversal guard; alternatively make `{name}` greedy. Pick
whichever keeps the openapi contract honest and the guard intact.

## Verify by hand after the deploy (operator)

Fetch `runs/snapshots/step-0001.yml` for the run above and open the presigned URL; then
fetch a name that is not in the listing and confirm `404 artefact_not_archived`.

## Blocked by

- (nothing)
