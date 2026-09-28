---
id: cqc-fe-09-technical-tab-and-reportless-runs
type: feat
status: in-progress
priority: 3
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [cqc-fe-07-drawer-navigation-runs-sharing]
attempts: []
---
# Engineers debug a run from its Technical tab, even when the run produced no report

Provenance and diagnostics in the run drawer (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 44, 45, 47).

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

- **Technical tab** showing:
  - launch, profile, dependency mode, agent;
  - cost, duration;
  - runner, with an uncommitted-changes flag (`git_dirty`);
  - code version, prompt hash;
  - extraction and navigation stats;
  - the raw report and the metadata JSON.
- **"Not recorded for this run":** the fields a run did not record are listed on
  one line, instead of a grid of `N/A` tiles.
- **Runs without a report:** the drawer still shows its Runs tab (when the LO
  has several runs) and its Technical tab, so staff can diagnose why it
  produced nothing. Checks and Evidence explain that there is no report.

## Acceptance criteria

- [ ] With a full fixture, every Technical field renders, and the raw report and metadata JSON are viewable.
- [ ] With a fixture missing several fields, they appear together on the single "Not recorded for this run" line and nowhere as `N/A` tiles.
- [ ] A run with `report: null` renders the Technical tab and, for a multi-run LO, the Runs tab. The other tabs say no report was produced.
- [ ] `git_dirty: true` shows the uncommitted-changes flag with a word, not colour alone.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and only the conventions this change triggers (routing tables in `implementation.md` / `frontend.md`).
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-07-drawer-navigation-runs-sharing
