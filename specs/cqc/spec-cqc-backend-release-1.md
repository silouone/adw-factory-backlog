# Spec: CQC backend, release 1 (read API)

Status: agreed 2026-09-26. Decisions: [`decisions-2026-09-26.md`](decisions-2026-09-26.md) (`BE-n`).
Deferred work: [`todo.md`](todo.md). Coding rules: `.agents/skills/cqc-guideline/`.

[`backend/openapi.yaml`](../../backend/openapi.yaml) is the **canonical API contract** (BE-29).
This document records the release-1 design and implementation context. The CMC side must point
to OpenAPI, generate its types from it, and retire its copy at
`domain-content-content-management-console/docs/cqc/backend-contract.md`.

## Problem

The CMC "Content quality" page (FE spec `docs/cqc/spec-cqc-fe-release-1.md` in the CMC repo)
needs an authenticated API to list checked LOs, show one run's report and evidence, and later
trigger checks. Today the checker's output sits in a private S3 bucket and nothing serves it.

## Release 1 in one picture

```
CMC (staff JWT) ─HTTPS─▶ HTTP API ─▶ authorizer λ ──GET /user/account/current──▶ Go1 API
                                 └─▶ api λ ──read──▶ DynamoDB (RUN#, LO#)
                                           └─read─▶ stage bucket (run-metadata, report, artefacts → presigned URL)

import-run ──copy via redaction gate──▶ stage bucket output/<lo>/<run>/run-metadata.json (written last)
                                                  │ S3 ObjectCreated
                                                  ▼
                                         projector λ ──▶ DynamoDB RUN#<run>, LO#<lo>
                                                    └─GET──▶ Go1 content API (mTLS): exists? provider_id
EventBridge rate(1 hour) ─▶ refresher λ ──list──▶ go1-scormassets/<env>/<asset_portal>/<lo>/
                                        └──────▶ DynamoDB LO#: current_revision, revision_changed, connected
```

The queued `POST /cqc/checks` acceptance, observed transition route, Batch submission,
and reviewer resolution routes are present. The Batch/Fargate runtime remains separate
release-2 work. See `todo.md`.

## Resources per stage (`dev`, `staging`, `prod`) — fully isolated (BE-13, BE-16)

| Resource | Name |
| --- | --- |
| Artefact bucket | `content-quality-checker--s3-artefacts-{stage}`, same key layout as the legacy bucket (`navigation_artefacts/{runs,output}/<lo>/<run>/`) |
| Table | `content-quality-checker--dynamodb-runs-{stage}` |
| Roles | `content-quality-checker--role-{api,projector,refresher,runner}-{stage}` (fixed names, BE-36) |
| SSM | `/credentials/{stage}/content-quality-checker/*` — `go1-root-jwt`, `go1-client-cert`, `go1-client-key`, `go1-ca-cert` (SecureString) for read API; `go1-client-id` and `go1-client-secret` (String or SecureString for existing stage compatibility; prefer SecureString for new values) for the runner's minted learner JWT; `staff-allow-list`; `config` JSON (defaults and limits, including `default_portal_id`) |
| Go1 | `prod` → Go1 prod; `dev`, `staging` → Go1 QA (BE-15) |
| SCORM assets | `prod` → `go1-scormassets/production/`; `dev`, `staging` → `go1-scormassets/qa/` (BE-35) |
| CORS | `prod`: `https://staff-new.go1.co` (to confirm). `staging`: `https://staff-new.qa.go1.cloud`. `dev`: that plus `http://localhost:3000`. |

Legacy `s3://content-quality-checker/` is read-only and only read by `import-run` (BE-16).

## Auth (BE-14)

- REQUEST authorizer on every route. Reads `Authorization: Bearer <jwt>`.
- Calls Go1 `GET {GO1_API_URL}/user/account/current` with that header.
  - Go1 non-2xx or missing header → `401 { error: "invalid_token" }`.
  - `roles` lacks `"Admin on #Accounts"` → `403 { error: "not_staff" }`.
  - Allow-list in SSM non-empty and user id absent → `403 { error: "not_staff" }`.
- Context passed to handlers: `{ user_id }` (from the Go1 response `id`). Logged on every request;
  stored as `triggered_by` on writes (release 2).
- Result cached 5 min keyed by `sha256(token)`. The token itself is never logged or stored.

## Data model (DynamoDB, one table)

| Item | Key | Holds |
| --- | --- | --- |
| Run | `PK=RUN#<run_id>`, `SK=RUN` | the `RunSummary` fields below + S3 keys of `run-metadata.json` and the report |
| LO aggregate | `PK=LO#<lo_id>`, `SK=LO` | `ContentRow` fields: latest / active / last failed run ids, `first_detected_at`, `last_run_at`, `run_count`, `asset_portal_id`, `current_revision`, `revision_changed`, `connected`, `content_hosts`, `authoring`, `exists_in_go1`, `provider_id` |
| LO → runs | GSI `lo-runs`: `lo_id` + `captured_at` | run history, newest first |
| List | GSI `list`: constant `_list=LO` + `sort_key` (`<status_rank>#<inverted last_run_at>`) | the page list in the FE's order without a scan |

Designed for thousands of LOs (BE-4); release 1 volume is ~35. Filters other than status apply
in the handler after the GSI query while volume is small — revisit when `total` passes ~2,000.

## Writers

**Projector** (S3 `ObjectCreated` on `navigation_artefacts/output/*/*/run-metadata.json`):

1. Read `run-metadata.json`; validate the fields it uses (fail with the key in the message).
2. Look for `navigation_artefacts/runs/<lo>/<run>/playability-report.json`; validate with
   `assertPlayabilityReport` from `poc0/observability/playability-report.mjs`.
3. Write `RUN#`: `lifecycle_status: "succeeded"` if the report exists and is valid; otherwise
   `"failed"` with `status_reason: "report_missing"` or `"report_invalid: <message>"` (BE-30).
4. Recompute `LO#` from all runs of that LO (query `lo-runs`) without replacing a cached
   `exists_in_go1` / `provider_id` value.
5. Call the Go1 content API with runtime SSM credentials. Found → cache
   `exists_in_go1: true` and `core.provider_id`; 404 → cache `exists_in_go1: false` and
   `provider_id: null`; any other read/decode failure leaves the cache unchanged and logs only
   `{ lo_id, reason }`. The Go1 gateway agent verifies the server certificate against Node's
   public trust roots (BE-22).
6. Idempotent: replaying the same event rewrites the same items.

**Refresher** (EventBridge `rate(1 hour)`, and per LO after a projector write):

1. For each `LO#` with an `asset_portal_id`: list `go1-scormassets/<env>/<asset_portal_id>/<lo>/`
   (delimiter `/`), keep numeric prefixes, `current_revision = max`.
2. `revision_changed = current_revision !== latest_run.go1_asset_revision` (both known), else `null`.
3. Classify the latest revision's listing as `connected` (proxy-shell) or static, porting
   `scripts/classify-scorm.sh`.
4. Missing grant or missing prefix → leave the fields `null` and log `{ lo_id, asset_portal_id, reason }`.

**Expected on dev and staging:** every legacy run navigated a Go1 **prod** LO, but dev and staging read Go1 QA and `go1-scormassets/qa/`. For example, `qa/10670560/10672378/` is empty (checked 2026-09-26). So on those stages, imported runs have `provider_id`, `current_revision`, `revision_changed` and `connected` all `null` by design. Enrichment is tested on prod, or on an LO confirmed to exist in QA.

**import-run** (CLI in `backend/`, run by an engineer):
`import-run --stage <stage> <lo_id>/<run_id>` copies both prefixes from the legacy bucket through
the same redaction re-scan as `scripts/context-extract/publish-context-packs.mjs` (`scanText` /
`redact`). That module calls `main()` at import time (line 876) and shells out to `aws --profile`.
So the scan must first move into a side-effect-free module that both the publisher and
`backend/` import (first build step in `todo.md`). `import-run` records the legacy object's
`LastModified` for the `captured_at` fallback, uploads
`run-metadata.json` last, and stamps `origin: "import"` on the projected row. Aborts on any
surviving credential; never overwrites an existing prefix without `--force`.

## Field mapping from `run-metadata.json` (1.1.0)

| API field | Source |
| --- | --- |
| `run_id`, `lo_id`, `environment`, `duration_seconds` | same names |
| `captured_at` | `captured_at` when set. Otherwise the UTC `YYYYMMDD-HHMMSS` stamp in `run_id`: every legacy id has one (`local-nav-<lo>-20260923-175123-<hex>`, `agent-nav-go1-20260924-082157-<hex>`). Otherwise the **legacy** object's `LastModified`, recorded by `import-run` before the copy. Never the stage bucket's `LastModified`, which is the import time. The API contract says never null. |
| `cost_usd`, `agent_model` | `agent.cost_usd`, `agent.model` |
| `git_dirty` | `code_version.git_dirty` |
| `launch_mode`, `tracking` | `launch.mode`, `launch.tracking` |
| `asset_portal_id` | `launch.configuration` (BE-32) |
| `package.vault_uuid` | `package.uuid` |
| `package.go1_asset_revision` | `package.version` (BE-32). `package.revision` is a Rustici constant and is not exposed as a revision. |
| `playability_status`, `claim_strength` | same names |
| `authoring` | `authoring.{tool, embedded_tools}`; absent on pre-1.1 records → `null` |
| `checks_summary` | from the report's `checks[]`: `{ check_id, criterion, status }`, ids unchanged (BE-28) |
| `origin` | `"import"` when written by `import-run`, else `"backend"` |

## Types

The following TypeScript sketch provides context; OpenAPI is authoritative for API fields and nullability.

```ts
type Lifecycle = "queued" | "provisioning" | "preflight" | "running" | "finalising"
  | "succeeded" | "failed" | "preflight_failed" | "timeout" | "cancelled";   // BE-23
type Playability = "Pass" | "Needs-review" | "Fail-block";                    // consumers keep unions open

interface RunSummary {
  run_id: string;
  lo_id: string;
  origin: "backend" | "import";
  lifecycle_status: Lifecycle;
  status_reason: string | null;            // required on failed | preflight_failed | timeout | cancelled
  playability_status: Playability | null;  // null unless succeeded
  claim_strength: "strong" | "conditional" | "limited" | null;
  environment: "local" | "e2b" | "fargate" | null;
  git_dirty: boolean | null;
  captured_at: string;                     // ISO 8601 UTC, never null
  duration_seconds: number | null;
  cost_usd: number | null;
  agent_model: string | null;
  launch_mode: "preview" | "launch" | null;
  tracking: "true" | "false" | null;
  portal_id: string | null;                // portal the check launched in (36743531 by default)
  asset_portal_id: string | null;          // partner portal of the SCORM asset (BE-32)
  package: { vault_uuid: string; go1_asset_revision: string } | null;
  provider_id: string | null;
  authoring: { tool: string; embedded_tools: string[] } | null;
  content_hosts: string[] | null;
  external_hosts: string[] | null;
  checks_summary: { check_id: string; criterion: string; status: string }[] | null;
  triggered_by: string | null;             // null in release 1
  batch_id: string | null;                 // null in release 1
  progress: null;                          // release 2
}

interface ContentRow {
  lo_id: string;
  exists_in_go1: boolean | null;             // true=found, false=404, null=not attempted/unknown
  provider_id: string | null;
  asset_portal_id: string | null;
  current_revision: string | null;         // max numeric <rev> in go1-scormassets (BE-34)
  revision_changed: boolean | null;        // null when either side is unknown
  latest_run: RunSummary | null;           // latest succeeded run
  active_run: null;                        // always null in release 1 (BE-23)
  last_failed_run: null;                   // release 2 (BE-23)
  first_detected_at: string | null;
  last_run_at: string | null;
  run_count: number;
  case_state: "open" | "resolved" | "none";  // run-derived or reviewer-resolved
  authoring: { tool: string; embedded_tools: string[] } | null;
  connected: boolean | null;               // package classification (BE-37)
  content_hosts: string[] | null;
  external_hosts: string[] | null;
  resolution: Resolution | null;           // active or last inactive reviewer decision
}
```

## Endpoints (release 1)

The endpoint table is a summary. OpenAPI defines parameters, response bodies, status codes and errors.

All under `/cqc`. Errors: `{ error, message?, request_id }`; `request_id` is the API Gateway request id.

| Method + path | Response |
| --- | --- |
| `GET /cqc/config` | `{ default_portal_id: "36743531", portals: [{id, name}], launch_modes: [{id:"preview", enabled:true}, {id:"launch", enabled:false, disabled_reason:"Blocked on analytics exclusion (B2)"}], agents: [...], estimate: {cost_usd_per_run, duration_minutes:{min,max}}, limits: {max_lo_ids_per_batch:20, max_batch_cost_usd:50, concurrency:5}, verdict_scope: "navigation-only" }` from SSM `config` (BE-7, BE-9, BE-24, BE-25). |
| `GET /cqc/content` | Query: `status` (repeatable: `Fail-block`, `Needs-review`, `Pass`, `running`, `failed_to_run`), `environment`, `authoring_tool`, `connected` (repeatable), `case_state`, `q` (LO id or run id), `offset` (default `0`), `limit` (default `100`; positive integers above `100` clamp to `100`). Repeats are OR within a field and fields are ANDed. `case_state=open` matches stored `open` and `none`; `none` and `resolved` match exactly. Order: Fail-block, failed_to_run, running, Needs-review, Pass; then `last_run_at` desc. Returns `{ items: ContentRow[], total, offset, limit, summary: { los_checked, by_status: { "Fail-block", "Needs-review", Pass }, running, failed_to_run, open, resolved, revision_changed, partners_affected }, facets: { status, environment, authoring_tool, connected, case_state } }` (BE-38). `total` counts filtered rows before pagination; `summary` and `facets` use the unfiltered rows. Facets retain stored case states and omit null values. `open` counts rows whose `case_state` is not `resolved`, so `open + resolved = los_checked`; `resolved` includes manual decisions. `revision_changed` counts only `true`. `partners_affected` counts distinct non-null `provider_id` values among open rows; it is `null` when all open-row providers are unknown, or `0` when there are no open rows. |
| `POST /cqc/content/{lo_id}/resolution` | `{ outcome, note }` records a reviewer decision and returns `202 Resolution` with `by`, `at`, latest `run_id`, `active: true`, and `reopened_by_run_id: null`. `fixed_verified` requires the latest succeeded run to be Pass. A latest run is required for every outcome. |
| `DELETE /cqc/content/{lo_id}/resolution` | Returns `200 { reopened: boolean }`; false when no active resolution exists. |
| `GET /cqc/content/{lo_id}/runs` | `{ items: RunSummary[] }`, newest first. `404 lo_not_found` if no run. |
| `GET /cqc/checks/{run_id}` | `{ run: RunSummary, metadata: RunMetadata \| null, report: PlayabilityReportV1 \| null, artefacts: [{ name, content_type }] }`. `metadata` and `report` are the S3 documents unchanged; `artefacts` from a `ListObjectsV2` on the run's prefixes. |
| `GET /cqc/checks/{run_id}/artefacts/{name}` | `{ url, expires_at, content_type }`, fresh presigned GET (15 min). `404 artefact_not_archived` when absent. `name` must appear in the run's listing — no path traversal. |

`provider_id` and `exists_in_go1` come from the Go1 content API (BE-11), fetched by the projector
and cached on `LO#`; the read path does not call Go1 per request. Operational lookup failures keep
the prior cached values (or `null` when none exist); only an observed 404 produces `false`.

Reviewer resolutions are stored on the LO item with unlisted `RES#<at>` history
items. A newer succeeded Fail-block run reopens an active decision in captured-at order.
Needs-review, crashed runs, and revision changes leave it active. The last inactive decision
remains in `GET /cqc/content` until replaced.

## Hard rules

- Never a public read path: the artefacts contain Content Partner course text.
- No writer reaches a CQC bucket without the redaction gate (BE-17).
- The backend never writes SCORM status and never turns a missing field into a Pass.
- The root JWT is used for GETs only.
- Release-1 rows are `succeeded` or `failed`; nothing fakes `queued`/`running`.

## Verification (release 1 done when)

1. `import-run --stage dev <lo>/<run>` for 3 legacy runs lands them in the dev bucket; the
   redaction re-scan passes; the projector writes `RUN#` and `LO#`.
2. `GET /cqc/content` from the CMC on `staff-new.qa.go1.cloud` with a staff JWT returns those LOs
   in the FE order; a non-staff JWT gets `403 not_staff`; no token gets `401 invalid_token`.
3. `GET /cqc/checks/{run_id}` returns metadata + report byte-identical to S3.
4. An artefact URL opens a snapshot; a cited-but-missing artefact returns `404 artefact_not_archived`.
5. A run copied without its report shows as `failed` / `report_missing`.
6. Once the DevOps grant exists, **on prod** (or on dev with an LO confirmed in `go1-scormassets/qa/`): `current_revision` and `connected` are filled for an imported LO.
7. Unit tests (Vitest) cover the mapping, list ordering, authorizer outcomes, and projector idempotence.
