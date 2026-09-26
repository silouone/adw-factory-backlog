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
