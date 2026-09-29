---
id: cmc-pending-01-pending-tab-filters-7f6f32
type: feat
status: queued
priority: 2
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: []
---
# Reviewers find one LO in the Pending tab by LO ID, title or provider

Servicedesk request. The CMC **Pending** tab has no filters. The queue holds more than 2,500
items globally and about 1,900 for Coursera, and reviewers scroll through it to find the one
LO they need to reject. They asked for LO ID and Title first, then Provider. This ticket is
frontend only: the API already supports all three filters. A submission date range is
**out of scope** (see the end of this ticket).

## Where it lives (read first)

- Both entry points render **`src/components/FlagList/index.tsx`**, so changing that one
  component fixes both:
  - global: `/content?status=pending` → `src/app/ContentList/index.tsx`
    (`FlagList allowBulkActions={false}`);
  - CP portal: `/portal/:id` → Content → `src/app/Portal/Content/index.tsx`
    (`FlagList allowBulkActions={true}`).
- Pending (`FlagList`) and Live (`PublishedContentList`) call the same endpoint through the same
  formatter: `LO.getPremiumMarketplace` → `LO.formatParams` (`src/services/LO.service.ts`,
  around line 520) → `GET /explore/lo`. `formatParams` already maps every filter needed here.
  The filters are missing only because `FlagList` never reads those query keys
  (its params are built in `FlagList.render`, around lines 355–363, from `offeringStatus`,
  `published`, `types` and `sort` only).
- **Live is the reference implementation. Copy it; do not invent a new approach.** Live builds
  these params in `src/components/PublishedContentList/index.tsx` (around lines 1407–1447) and
  defaults the search toggles around lines 439–463.

| Filter | URL query key (same as Live) | `BrowseParams` field |
| --- | --- | --- |
| Title / text search | `q` + `title_enabled`, `description_enabled`, `summary_enabled`, `learning_outcomes_enabled` | `keyword` + `searchToggles` |
| LO ID | `lo_id` | `id: [lo_id]` |
| Provider | `provider` (facet `instance`) | `providers` |

## What to build

**Params (in `FlagList.render`).** Read `q`, `lo_id`, `provider` and the four `*_enabled`
toggles from `history.location.query` and add them to `params`:

```ts
keyword: q,
searchToggles: { title_enabled, description_enabled, summary_enabled, learning_outcomes_enabled },
id: lo_id ? [lo_id] : undefined,
providers, // global page only
```

- **The toggles are required.** `formatParams` sends `keyword` only when `searchToggles` is
  set. Copy Live's rule that turns all toggles on by default. Without it a title search silently
  does nothing.
- **Use the same query keys as Live.** `history.push` merges queries (`src/utils/history.ts`,
  around lines 78–86) and the tab bar pushes only `{ status }`, so a search typed on Live carries
  over to Pending. That is fine, but the Pending filter bar must **show** the values it inherits
  (through `initialValues`), so it never filters without the reviewer seeing why.

**Filter row** in `FlagList`, next to the existing Status select. Reuse Live's pieces; no new
Go1d components:

- `FilterNav` (`src/components/FilterNav`) as the form. It already pushes values to the URL on
  change and submit.
- `SearchBar` plus the four toggle `Checkbox`es (Title / Description / Summary / Outcomes).
- An **LO ID** field (`Field` + `TextInput`, `name="lo_id"`).
- **Provider** (`FacetFilter` on `LO.getFacet`, facet `instance`), labelled **"Provider"**. Not
  "Product": on Live, "Product" means Premium vs Marketplace. Show it **only on the global
  page** (when `match.params.id` is not set). On the portal page `formatParams` forces
  `provider = [instanceId]`, so a Provider filter there would do nothing. Scope the facet
  request to pending content (`offeringStatus: "pending"`, `published: 10`) so the dropdown
  lists only providers that have something pending.
- Update `clearSearchFilters` so it also clears `q`, `lo_id` and `provider`.

**Bulk actions (portal page): intended behaviour change.** "Approve all" and "Reject all" go
through `BulkContentActionModal`, which receives the same `params` and re-queries
`LO.getPremiumMarketplace` to resolve "all" (`src/components/BulkContentActionModal/index.tsx`,
around lines 205–208). Once the filters are in `params`, **"Reject all" acts on the filtered
set**, which is what the requester wants: filter to one LO ID, then reject it. In
`src/components/BulkActionModal/index.tsx`, the "all"/inverted path pages through
`browse({ ...params, offset, limit })` and keeps `id`, `keyword` and `provider`. The
explicit-selection path (`getItemsWithSelectedLoIds`) replaces `id` with the checked rows,
which already come from the filtered list. The prefix row's `N pending` comes from the filtered
total, so the scope stays visible.

## Acceptance criteria

- [ ] `?lo_id=123` sends `id: ["123"]` to `LO.getPremiumMarketplace` and the LO ID field shows `123`.
- [ ] `?q=python` with no `*_enabled` keys sends `keyword` with every search toggle on (Live's default rule).
- [ ] `?q=python&title_enabled=true&description_enabled=false…` sends exactly those toggles, and the checkboxes show them.
- [ ] On the global page the Provider filter renders, its facet request is scoped to pending content, and `?provider=x` sends `providers: ["x"]`.
- [ ] On the portal page (`match.params.id` set) the Provider filter does not render.
- [ ] Clearing filters removes `q`, `lo_id` and `provider` from the URL.
- [ ] The bulk action modal receives the filtered `params` (`id`, `keyword`, `searchToggles`).
- [ ] The existing `FlagList` tests still pass. The existing Status select and sort behave as before.

## Constraints

- Write the new cases in `src/components/FlagList/index.test.tsx` (6 cases today) first and see
  them fail, then implement.
- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and
  the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not
  edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Follow `.agents/conventions/implementation.md`, `.agents/conventions/frontend.md` and
  `.agents/conventions/references/testing.md`, plus the table recipe. Test pitfalls are listed in
  `.agents/overlay/TESTING-NOTES.md`: passive effects do not flush after RTL `render`/`fireEvent`,
  so use the flush wrapper documented there.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d offline: check components and props against the installed `@go1d/go1d` type definitions.
- Touch only what the filters need. The side findings below stay out of this PR.

## Out of scope

- **Submission date range.** `formatParams` has no `created[min]/[max]` or submission-time
  parameter, and the Pending table's `created` sort is LO creation, not submission. First confirm
  with domain-search-explore that the index can filter on a submission or status-change
  timestamp by range. That goes in a separate ticket.
- **Side findings. Report them in the PR description; do not fix them here:**
  1. The Pending "Status" select does not filter: `flag_type` goes into the URL but never into `params`.
  2. A possible crash: with both Status options selected, `flag_type` is an array, and
     `performApproveAction` / `performRejectAction` call `flagType.toLowerCase()`.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker
`node:14.21.3`. By hand after merge: QA on `/content?status=pending` and
`/portal/:id/content?status=pending`.
