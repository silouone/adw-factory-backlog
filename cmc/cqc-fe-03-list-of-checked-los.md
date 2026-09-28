---
id: cqc-fe-03-list-of-checked-los
type: feat
status: in-progress
priority: 1
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-02-gated-content-quality-page]
attempts: []
---
# Staff see every checked LO with its latest verdict and check mini-strip

The read path of the Content quality list (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 3, 4, 6–9 and 22; sections "Service boundary", "API consumed",
"Key type shapes", "Domain rules", "Go1 enrichment"). The API is a draft
contract in `docs/cqc/backend-contract.md`, and there is no live backend yet,
so everything is tested against a mocked service.

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
- **This ticket adds the first real content under the root:** the CQC service reads its base URL from the root's `apiBaseUrl` prop (never from `window.GO1` directly). When `apiBaseUrl` is missing the root already shows the not-configured message, so do not fetch at all in that case; the service's own not-configured error covers a runtime misconfiguration.

## What to build

- **A plain typed CQC service** with its own axios instance on the CQC base
  URL. It sends the signed-in staff user's own Go1 JWT (the current user's
  `jwt`) as a bearer token. It exposes `GET /cqc/content` (repeatable filters
  `status`, `environment`, `authoring_tool`, `connected`; plus `case_state`,
  `q`, `offset`, `limit`) and returns typed DTOs (`ContentRow`, `RunSummary`,
  `summary`, `facets`). It is consumed through the existing `useQuery` hook,
  not the normalized entity layer. It must not import component-owned types.
- **A pure domain module** owned by the feature, tested directly:
  - journey order: launch, navigation, completion, runtime, media, resume; unknown criteria go last;
  - check labels (e.g. `completion-recorded` → "Completion is recorded"), with unknown ids humanised and never shown raw;
  - status vocabulary: Pass → "passed", Fail-block → "failed", Needs-review → "not verified", anything else → "Unknown status: X", with open-ended unions.
- **The list** on the existing generic table:
  - one row per LO, ordered Fail-block, then check failed to run, then running, then Needs-review, then Pass, each by last run;
  - a verdict pill with icon and word, where Needs-review is neutral and reads "not verified";
  - a six-icon mini-strip in journey order, where each icon has an accessible name such as "Completion is recorded: failed";
  - pagination through `offset`/`limit`;
  - each row opens from a real `<button>`; the drawer itself arrives in a later ticket, so the button can be inert or navigate to a `lo` URL param.
- **States:** loading, empty ("No content has been checked yet"), and errors
  with distinct messages for not configured, 401 (session rejected), 403
  (missing staff role) and other failures, each with Retry.
- **Titles** from the existing content-gateway service, batched per visible
  page. A missing record shows `N/A` and the row stays.

## Acceptance criteria

- [ ] A service test asserts the exact path, query params (repeatable filters) and the `Authorization` header on a mocked axios instance.
- [ ] Domain-module tests cover journey order with an unknown criterion, the humanised unknown label, and the four status-vocabulary cases.
- [ ] A page test with the service mocked shows rows in urgency order and mini-strip accessible names in journey order.
- [ ] 401, 403 and 500 each show their own message plus a Retry button that refetches.
- [ ] Loading and empty states render.
- [ ] A title missing from content-gateway renders `N/A`.
- [ ] Status is never conveyed by colour alone: every status has an icon and a word.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md`, `.agents/conventions/references/data-and-state.md` and the
  table recipe before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Go1d: verify every component and prop against the installed `@go1d/go1d`
  version. If the Go1d source repository is unreachable from the sandbox, use
  the installed package's type definitions and state the exception to the
  go1d-evidence rule in the PR description.
- No invented data: any field the backend cannot provide shows `N/A`.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-02-gated-content-quality-page
