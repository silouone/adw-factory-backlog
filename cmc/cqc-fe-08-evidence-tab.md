---
id: cqc-fe-08-evidence-tab
type: feat
status: in-review
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-06-run-drawer-header-and-checks]
attempts: [{"runId":"cqc-fe-08-evidence-tab-1790611451518","branch":"adw/cqc-fe-08-evidence-tab","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-08-evidence-tab-1790611451518/workspace","outcome":"in-review","provider":"codex","model":"gpt-5.6-sol","pr":"https://github.com/go1com/domain-content-content-management-console/pull/142"}]
---
# The Evidence tab shows every proof behind a verdict, ranked by trust, and explains each gap

The evidence view of the run drawer (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 38–43 and 57; "Domain rules: evidence availability",
"Missing-evidence reasons", "Data fetching: downloads").

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

- **Evidence availability, a pure domain rule tested directly.** For each
  evidence item cited by the report, work out:
  - available or unavailable;
  - the resolved file;
  - the viewer kind: SCORM table, snapshots stepper, image, text or raw;
  - the trust rank, 1–8, most direct first.

  Accessibility snapshots are added as an extra item. The agent's own account
  (the narrative) has the lowest trust and is labelled that way. A narrative
  whose cited file is missing falls back to the agent's last message, with a
  note saying so.
- **Missing-evidence reasons:** one explanation per evidence kind, covering why
  it is missing and what would make it available.
- **Artefacts on demand:** `GET /cqc/checks/{run_id}/artefacts/{name}` returns
  `{ url }`, a fresh presigned URL. It is fetched only when a viewer opens and
  never cached across sessions. Downloads check the HTTP status. A cached
  failure offers Retry, which asks for a fresh URL.
- **Evidence tab**, titled "Evidence (N of M available)":
  - a trust-ordered list with one banner stating the archive gap;
  - viewers:
    - SCORM API trace as a table (calls, writes, last status write), parsed defensively;
    - an accessibility snapshot stepper;
    - the final screenshot (image);
    - text and raw.
- **Unavailable evidence** is greyed out with ⊘ but still clickable. It opens a
  panel with N/A, the cited file, its trust rank, why it is missing and what
  would make it available.
- **Pinned limitations:** on tabs other than Checks, the limitations are
  pinned as a one-line link back to them.
- Checks in the Checks tab link to their evidence inline, reusing the same viewers.

## Acceptance criteria

- [ ] Domain tests cover availability, the resolved file, trust ordering, the added snapshot item and the narrative fallback note.
- [ ] Clicking an unavailable item opens the N/A panel with its kind-specific reason.
- [ ] A SCORM trace fixture renders as a table with the last status write identified. A malformed trace renders a readable error, not a crash.
- [ ] A failed or expired download shows Retry. Retry requests a fresh URL from the service and renders the content.
- [ ] The snapshot stepper steps forward and back with labelled buttons.
- [ ] The tab title shows "N of M available".
- [ ] The limitations link is pinned on the Evidence tab and absent on Checks, which shows them in full.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and only the conventions this change triggers (routing tables in `implementation.md` / `frontend.md`).
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).
- Never cache presigned URLs beyond the session.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-06-run-drawer-header-and-checks
