---
id: cqc-fe-07-drawer-navigation-runs-sharing
type: feat
status: in-progress
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-06-run-drawer-header-and-checks]
attempts: []
---
# Staff step through LOs, compare runs and share a stable link from the drawer

Drawer navigation, history and sharing (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 29, 34, 46, 48, 49; "Page shape: stable run links").

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
- **Drawer focus is built in (cqc-fe-01):** changing the drawer's `contentKey` (for j/k navigation) already moves focus to the heading identified by `headingId`.

- **From cqc-fe-03 (#137), reuse, don't duplicate:**
  - Service `src/services/ContentQuality.service.ts`: `new ContentQualityApi(apiBaseUrl, jwt)`, `listContent(params)`, typed DTOs (`RunSummary`, `ContentRow`, `ContentSummary`, `ContentFacets`, `ContentListResponse`, `ContentListParams`) and `ContentQualityServiceError`. Add new endpoints as methods here.
  - Pure domain `src/components/ContentQuality/domain.ts`: `JOURNEY_ORDER`, `sortChecksByJourney`, `getCheckLabel`, `getStatusLabel`, `getContentUrgency`, `sortContentRows`. Extend it for new domain rules; test it directly (`domain.test.ts`).
  - List `src/components/ContentQuality/CheckedContentList/` (table, `columns.tsx`, `PAGE_LIMIT = 20`) and `common/formatMissing.ts` (the `N/A` rule).
  - URL state `CheckedContentList/useCheckedContentListParams.ts` reads `offset` and `lo`. ⚠️ Today `onPageChange` pushes `{ offset }` and `onOpen` pushes `{ lo }`, each **replacing the whole query**. Any ticket that adds URL state (filters, `run`, drawer params) must first make these pushes **merge** into the current query, with a test that paging keeps the filters and opening a row keeps the page.

## What to build

- **Step through the queue:** j/k keys and ↑/↓ buttons move to the next or
  previous LO in the current list order, keeping the active tab. A position
  indicator reads "3 of 27", with a breadcrumb.
- **Run picker** in the header, only when the LO has more than one run. It
  selects a past run.
- **Runs tab**, only with more than one run: the run history from `GET /cqc/content/{lo}/runs`.
- **Stable run links:** a `run` URL param opens that specific run.
  "Copy link to this run", in the sticky footer, copies a URL built on `run`.
- **"Open in player"** in the header reuses the CMC's existing one-time-token player flow.
- **Banners** in the drawer:
  - a newer attempt failed to run, with a link to it;
  - a check is in progress, with a link to follow it;
  - revision changed;
  - simulated or reopened state.
  The run being read must never be mistaken for the latest truth.
- A keyboard hint in the sticky footer.

## Acceptance criteria

- [ ] With the Evidence tab active, pressing `j` opens the next LO with the Evidence tab still active and focus on the heading. `k` goes back.
- [ ] The ↑/↓ buttons do the same and are disabled at the ends. The position reads "N of M".
- [ ] j/k typed inside a text input do not navigate.
- [ ] The run picker and Runs tab are absent for a single-run LO and present for a multi-run LO; picking a run updates the `run` URL param.
- [ ] Loading a URL with `run=<id>` opens that run.
- [ ] "Copy link to this run" writes the run URL to the clipboard (mocked) and confirms visibly.
- [ ] Each of the four banners renders under its condition, with its link where specified.
- [ ] "Open in player" calls the existing one-time-token flow (mocked at the service boundary).

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and only the conventions this change triggers (routing tables in `implementation.md` / `frontend.md`).
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-06-run-drawer-header-and-checks
