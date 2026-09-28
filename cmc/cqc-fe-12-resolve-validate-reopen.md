---
id: cqc-fe-12-resolve-validate-reopen
type: feat
status: blocked
priority: 3
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-05-summary-tabs-search-filters, cqc-fe-06-run-drawer-header-and-checks]
attempts: [{"runId":"cqc-fe-12-resolve-validate-reopen-1790612934319","branch":"adw/cqc-fe-12-resolve-validate-reopen","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-12-resolve-validate-reopen-1790612934319/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"},{"runId":"cqc-fe-12-resolve-validate-reopen-1790621441878","branch":"adw/cqc-fe-12-resolve-validate-reopen-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-12-resolve-validate-reopen-1790621441878/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"}]
---
# Content ops resolve, validate and reopen an LO with an outcome and a note

The case workflow for release 1 (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 50–54; "API consumed": `POST` and `DELETE /cqc/content/{lo}/resolution`).
This is ship-plan step 3. The case workflow beyond resolve and validate is out of scope.

## Base and what already exists (read first)

- **Base branch: `cqc/release-1`**, not `master`. Your work branches from it and your PR
  targets it; the operator merges there. The whole page reaches `master` later in one PR for
  the CLB team. Never target `master`.
- **Already on `cqc/release-1`** (from cqc-fe-01 #135 and cqc-fe-02 #136):
  - Route **`/quality`** in `src/app/index.tsx` (production URL `/content/quality`). It is
    deliberately not `/content-quality`: the menu marks items active by path prefix, so that
    path would also light up the "Content" item. The "Content quality" left-menu item links to it.
  - Feature root **`src/components/ContentQuality/index.tsx`**:
    `ContentQuality({ apiBaseUrl?: string })` renders the page header "Content quality" and,
    when `apiBaseUrl` is missing, "Content Quality Checker API is not configured". The route
    passes it `CQC_API_URL`. Extend this root; do not create a second one or a second header.
  - Runtime config **`CQC_API_URL`** in `src/config.ts` (container `APP_CQC_API_URL` → nginx SSI
    → `window.GO1.CQC_API_URL`), normalised by `runtimeValue()`: `undefined` means not configured.
  - **No feature gate.** Statsig was dropped (spec D-1 amended 2026-09-28): the page is visible to
    every CMC user. Do not add any gating.
- **No live backend yet.** Test against a mocked CQC service. The API shape is
  `docs/cqc/backend-contract.md`; the backend's canonical spec (same paths and types) lives in
  the content-quality-checker repo and becomes `backend/openapi.yaml` later.
- **Tests in this repo** (`.agents/overlay/TESTING-NOTES.md`): passive effects do not flush after
  RTL `render`/`fireEvent`; use the `ReactDOM.render` flush wrapper it documents. TS 3.5 has no
  `keyCode` in `fireEvent` init objects (use `{ key: "..." }`). tslint enforces `prefer-for-of`.
- **Go1d offline:** the sandbox cannot reach the Go1d source. Verify components and props against
  the installed `@go1d/go1d` type definitions and state the CMC-RX-13 exception in the PR
  description; do not search the web or the disk for the Go1d repository.

- **From cqc-fe-03 (#137), reuse, don't duplicate:**
  - Service `src/services/ContentQuality.service.ts`: `new ContentQualityApi(apiBaseUrl, jwt)`, `listContent(params)`, typed DTOs (`RunSummary`, `ContentRow`, `ContentSummary`, `ContentFacets`, `ContentListResponse`, `ContentListParams`) and `ContentQualityServiceError`. Add new endpoints as methods here.
  - Pure domain `src/components/ContentQuality/domain.ts`: `JOURNEY_ORDER`, `sortChecksByJourney`, `getCheckLabel`, `getStatusLabel`, `getContentUrgency`, `sortContentRows`. Extend it for new domain rules; test it directly (`domain.test.ts`).
  - List `src/components/ContentQuality/CheckedContentList/` (table, `columns.tsx`, `PAGE_LIMIT = 20`) and `common/formatMissing.ts` (the `N/A` rule).
  - URL state `CheckedContentList/useCheckedContentListParams.ts` reads `offset` and `lo`. ⚠️ Today `onPageChange` pushes `{ offset }` and `onOpen` pushes `{ lo }`, each **replacing the whole query**. Any ticket that adds URL state (filters, `run`, drawer params) must first make these pushes **merge** into the current query, with a test that paging keeps the filters and opening a row keeps the page.

## What to build

- **Service:** `POST /cqc/content/{lo}/resolution` (outcome and note) and
  `DELETE /cqc/content/{lo}/resolution` (reopen).
- **Resolve…** on open LOs, on the row and in the drawer footer. It opens a
  modal owned by its trigger, with:
  - an outcome: fixed and verified, validated manually, false positive, or accepted as is
    (contract values `fixed_verified`, `validated_manually`, `false_positive`, `accepted_risk`);
  - a required note of at least 10 characters (the contract rejects shorter notes).
- **"Fixed and verified"** is selectable only when the LO's latest run passes.
  Otherwise it is disabled with the reason.
- **Validate** on passing LOs records a human confirmation through the same
  endpoint and modal, with the outcome `validated_manually` preselected.
- A `409 latest_run_not_pass` response (a newer run no longer passes) shows
  its reason in the modal and refetches the LO.
- **Resolution display:** who resolved it, when, the outcome and the note, in
  the drawer case line and on the Resolved tab.
- **Automatic reopen:** when `resolution.active` is false with
  `reopened_by_run_id`, the drawer shows the reopened banner linking to that run.
- **Reopen** on a resolved LO calls `DELETE` and moves it back to Open.
- On success: a confirmation, and the list and tab counts refetch.

## Acceptance criteria

- [ ] Submitting with an empty note or a note under 10 characters shows a validation error and sends nothing.
- [ ] A mocked `409 latest_run_not_pass` shows its reason in the modal and does not close it.
- [ ] "Fixed and verified" is disabled with its reason when the latest run is not Pass, and enabled when it is.
- [ ] A valid submit calls the resolution endpoint with the outcome and note, shows a success confirmation, closes the modal, returns focus to its trigger and refetches the list.
- [ ] A resolved LO shows who, when, the outcome and the note, and offers Reopen. Reopen calls `DELETE` and the LO returns to Open.
- [ ] Validate appears only on passing LOs.
- [ ] A reopened resolution shows the reopened banner with a link to the reopening run.
- [ ] Service tests assert exact paths and bodies.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md`, the modal recipe, `rules/react/modal-trigger-ownership.md` and `references/testing.md` before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-05-summary-tabs-search-filters
- cqc-fe-06-run-drawer-header-and-checks
