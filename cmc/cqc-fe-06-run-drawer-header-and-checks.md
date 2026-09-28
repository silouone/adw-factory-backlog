---
id: cqc-fe-06-run-drawer-header-and-checks
type: feat
status: done
priority: 1
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-01-drawer-keeps-keyboard-focus, cqc-fe-03-list-of-checked-los]
attempts: [{"runId":"cqc-fe-06-run-drawer-header-and-checks-1790603212292","branch":"adw/cqc-fe-06-run-drawer-header-and-checks","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-06-run-drawer-header-and-checks-1790603212292/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/140","provider":"codex","model":"gpt-5.6-sol"}]
---
# Clicking a row opens a run drawer that explains the verdict check by check

The core of the run drawer (spec `docs/cqc/spec-cqc-fe-release-1.md`, user
stories 28, 30–33, 35–38 and 57; "Drawer layout (drawer 1)", "Data fetching",
"Domain rules: default focused check"). The visual reference was the (unshipped) prototype's drawer 1; the spec describes it fully.

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
- **Drawer focus is built in (cqc-fe-01):** `Drawer` takes `headingId?: string` and `contentKey?: string | number`. Pass the id of the drawer's own heading element as `headingId` to get focus-on-open, the Tab trap, focus return to the opener, Escape/close/backdrop closing, and `role="dialog"` named by `aria-labelledby`. Pass the run id (or LO id) as `contentKey` so focus returns to the heading when the drawer switches to another item. Do not re-implement any of this.

- **From cqc-fe-03 (#137), reuse, don't duplicate:**
  - Service `src/services/ContentQuality.service.ts`: `new ContentQualityApi(apiBaseUrl, jwt)`, `listContent(params)`, typed DTOs (`RunSummary`, `ContentRow`, `ContentSummary`, `ContentFacets`, `ContentListResponse`, `ContentListParams`) and `ContentQualityServiceError`. Add new endpoints as methods here.
  - Pure domain `src/components/ContentQuality/domain.ts`: `JOURNEY_ORDER`, `sortChecksByJourney`, `getCheckLabel`, `getStatusLabel`, `getContentUrgency`, `sortContentRows`. Extend it for new domain rules; test it directly (`domain.test.ts`).
  - List `src/components/ContentQuality/CheckedContentList/` (table, `columns.tsx`, `PAGE_LIMIT = 20`) and `common/formatMissing.ts` (the `N/A` rule).
  - URL state `CheckedContentList/useCheckedContentListParams.ts` reads `offset` and `lo`. ⚠️ Today `onPageChange` pushes `{ offset }` and `onOpen` pushes `{ lo }`, each **replacing the whole query**. Any ticket that adds URL state (filters, `run`, drawer params) must first make these pushes **merge** into the current query, with a test that paging keeps the filters and opening a row keeps the page.

## What to build

- **Service:** `GET /cqc/checks/{run_id}` (returns `run`, `metadata`, `report`
  (playability report V1) and artefact names) and `GET /cqc/content/{lo}/runs`.
- **URL-driven drawer.** A `lo` URL param opens the latest run of that LO in
  the CMC `Drawer` at size md, using the focus behaviour from cqc-fe-01. The
  list stays visible behind it and the open row is highlighted. Esc, the close
  button or a backdrop click close it and return focus to the row button.
- **Stale-response protection.** Detail and history are fetched together and
  committed only if they still match the current selection.
- **Sticky header:**
  - "LO <id>" plus the Go1 title (`N/A` when unknown);
  - the verdict pill;
  - the claim strength with an explanation per level;
  - the context line: checked date · revision · launch mode with tracking · runner;
  - the checker summary, labelled as such, clamped to two lines with "More";
  - the case line: open/resolved, days open, first detected, run count, partner.
- **Check strip:** the six checks in journey order. Clicking a step scrolls
  that check to the top, expands it with its evidence inline, and collapses the
  others. It exposes current/pressed/expanded state.
- **Checks tab** (the default tab, in a properly marked-up tablist):
  - the limitations ("what the run could not observe") first, word for word and ungrouped;
  - then each check in journey order: its reasons word for word, severity,
    whether it counts toward the verdict, and declared vs observed values on demand.
- **Default focused check:** the first Fail-block, else the first non-Pass.
- The domain rule for the default focused check goes in the feature's pure
  domain module, with a direct test.

## Acceptance criteria

- [ ] Clicking a row button opens the drawer, sets `lo` in the URL, and puts focus on the drawer heading.
- [ ] Esc closes the drawer, clears the URL param and returns focus to that row button.
- [ ] Opening a run with a Fail-block check expands that check by default; with none, the first non-Pass check.
- [ ] Clicking a strip step expands that check with inline evidence and collapses the others, and its current state is exposed.
- [ ] Limitations render word for word above the checks.
- [ ] Stale-response test: open LO A, then LO B, resolve B then A. Only B renders.
- [ ] The summary clamps with "More" and expands on click.
- [ ] Domain test for the default focused check.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md`, `references/data-and-state.md`, the modal/information-display recipes and `references/testing.md`.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).
- No invented data: absent fields show `N/A`.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-01-drawer-keeps-keyboard-focus
- cqc-fe-03-list-of-checked-los
