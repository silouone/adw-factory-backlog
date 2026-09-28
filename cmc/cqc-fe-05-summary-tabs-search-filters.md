---
id: cqc-fe-05-summary-tabs-search-filters
type: feat
status: queued
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-03-list-of-checked-los]
attempts: []
---
# Staff narrow the Content quality list with cards, tabs, search and shareable filters

Backlog overview and filtering (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 5, 16–21).

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

## What to build

- **Summary cards** from the response's `summary`: Open, Fail-block, Revision changed, Partners affected (`N/A` when absent).
- **All / Open / Resolved tabs** with counts, mapped to `case_state`.
- **Search** by LO id or run id, mapped to `q`.
- **Multi-select dropdown filters in one row**: Status, Runner (`environment`),
  Authoring tool, Connected, Partner. They reuse the CMC's shared filter
  multi-select (Go1d `MultiSelect`, "Any" when empty, stays open while
  ticking). Each option shows its count from `facets`.
- **Partner filter** visible but disabled, with its reason ("Partner data is not available yet").
- **"Clear all filters"** whenever any filter, search or non-default tab is active.
- **Filtered-empty state** with "Clear all filters".
- **URL state**: every filter, the tab and the search live in the URL. Changing
  any of them resets `offset`, so a filtered view can be shared in Slack.

## Acceptance criteria

- [ ] A page test ticks two Status options: the service is called with both repeated `status` params and `offset=0`, and the URL reflects both.
- [ ] Loading a URL with filters pre-applied renders them as selected and calls the service with them.
- [ ] Option counts come from `facets`; an empty filter reads "Any".
- [ ] The Partner filter is disabled and its reason is reachable by assistive technology.
- [ ] "Clear all filters" resets the URL and the service params; it is absent when nothing is active.
- [ ] Filtered-empty shows its own message plus "Clear all filters".
- [ ] Tabs are properly marked up (tablist/tab, selected state) with counts.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md`, `references/data-and-state.md` and the table recipe before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-03-list-of-checked-los
