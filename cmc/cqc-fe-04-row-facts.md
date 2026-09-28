---
id: cqc-fe-04-row-facts
type: feat
status: in-review
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [cqc-fe-03-list-of-checked-los]
attempts: [{"runId":"cqc-fe-04-row-facts-1790603188103","branch":"adw/cqc-fe-04-row-facts","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-04-row-facts-1790603188103/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/139","provider":"codex","model":"gpt-5.6-sol"}]
---
# Each Content quality row tells staff how old, how stale and how connected the problem is

Row enrichment for the Content quality list (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 10–15; type shapes in the "Key type shapes" section).

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

On every list row, from the `ContentRow` and `RunSummary` fields the service already returns:

- **Authoring tool**: Rise 360, Storyline, Captivate, iSpring, Adapt, Lectora
  or Unknown, plus embedded tools. `null` authoring shows `N/A`.
- **Connected or Static**: "Connected (served by <host>)" when `connected` is
  true, listing `content_hosts`; "Static" when false; `N/A` when null.
- **Timing facts**: first detected, last run, days open, run count, runner (E2B or local).
- **"Revision changed, not re-checked"** when `revision_changed` is true.
- **"Check failed to run"** with its `status_reason` when `last_failed_run` is
  newer than `latest_run`. It is never styled as Fail-block.
- **Queued or running** with its progress, when `active_run` is set.

## Acceptance criteria

- [ ] A page test with mocked rows shows each fact above, including `N/A` for each null field.
- [ ] A row with `last_failed_run` reads "Check failed to run" with its reason and does not carry the failed/Fail-block styling or wording.
- [ ] A row with `active_run` shows its lifecycle status and progress.
- [ ] A row with `revision_changed: true` shows "Revision changed, not re-checked".
- [ ] Every status is shown with an icon and a word, never colour alone.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and only the conventions this change triggers (routing tables in `implementation.md` / `frontend.md`).
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).
- No invented data.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-03-list-of-checked-los
