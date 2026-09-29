---
id: cqc-fe-10-trigger-checks
type: feat
status: in-review
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-04-row-facts, cqc-fe-06-run-drawer-header-and-checks]
attempts: [{"runId":"cqc-fe-10-trigger-checks-1790612910173","branch":"adw/cqc-fe-10-trigger-checks","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-10-trigger-checks-1790612910173/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"},{"runId":"cqc-fe-10-trigger-checks-1790621407480","branch":"adw/cqc-fe-10-trigger-checks-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-10-trigger-checks-1790621407480/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"},{"runId":"cqc-fe-10-trigger-checks-1790632188642","branch":"adw/cqc-fe-10-trigger-checks-3","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-10-trigger-checks-1790632188642/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/146","provider":"codex","model":"gpt-5.6-sol"}]
---
# Staff paste LO ids from Slack to trigger checks, and re-run a check from a row or the drawer

Triggering checks (spec `docs/cqc/spec-cqc-fe-release-1.md`, user stories
23–26; "API consumed": `GET /cqc/config`, `POST /cqc/checks`, `POST /cqc/batches`).
This is ship-plan step 2.

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

- **Service:**
  - `GET /cqc/config`, which gives the trigger defaults, the estimates and the limits;
  - `POST /cqc/checks` for one LO and `POST /cqc/batches` for several; both return 202, and the batch response lists rejected ids with reasons.
- **Paste bar** above the list: one or more LO ids separated by commas,
  spaces or newlines, and a Check button. Before triggering, it shows a live
  estimate from `/cqc/config`: count, cost (about $2 per LO, from config),
  duration and parallelism. It warns about invalid ids and about a batch over
  the configured limit.
- **Submit:** one valid id → a single check; several → one batch. A toast
  says how many were queued and which were rejected and why (not a number,
  already running, over the limit).
- **Re-run** on each row and **"Run check again"** (primary) in the drawer
  footer. Both are disabled, with the reason, while that LO has a run in
  progress.
- If `/cqc/config` is unavailable, the paste bar and the re-run actions are
  disabled and say why. No estimate is invented.

## Acceptance criteria

- [ ] Typing "123, abc 456" shows a count of 2, flags "abc" as invalid, and shows the estimate computed from mocked config values.
- [ ] Exceeding the configured batch limit shows a warning and disables Check.
- [ ] A single valid id calls the single-check endpoint. Several call the batch endpoint once.
- [ ] A batch response with rejected ids shows a toast with the queued count and each rejected id and reason.
- [ ] Re-run on a row with an active run is disabled with its reason. On an idle row it triggers a single check.
- [ ] "Run check again" in the drawer behaves the same.
- [ ] A config failure disables the trigger actions with a visible reason.
- [ ] Service tests assert exact paths and bodies.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md`, `references/data-and-state.md` and `references/testing.md` before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).
- Costs and limits come from `/cqc/config` only; never hard-code them.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-04-row-facts
- cqc-fe-06-run-drawer-header-and-checks
