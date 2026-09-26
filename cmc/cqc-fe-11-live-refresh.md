---
id: cqc-fe-11-live-refresh
type: feat
status: queued
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [cqc-fe-04-row-facts, cqc-fe-06-run-drawer-header-and-checks]
attempts: []
---
# The list and the open drawer refresh themselves while a check is running

Polling (spec `docs/cqc/spec-cqc-fe-release-1.md`, user stories 14 and 27;
"Data fetching: polling").

## What to build

- While any run in the list or the open drawer is non-terminal (queued,
  provisioning, preflight, running, finalising), refetch the list and the
  open run every 10 seconds. Stop when every run is terminal. Restart when a
  new run appears, for example after a trigger.
- An **"In progress" strip** above the list shows the running checks with their progress.
- Refetching must not reset the drawer's tab, focused check or scroll, and
  must not steal focus.
- Polling stops when the page unmounts.

## Acceptance criteria

- [ ] With fake timers, a list containing an active run refetches at 10 s. When the mocked response makes all runs terminal, no further request happens.
- [ ] A list with only terminal runs never polls.
- [ ] The open drawer's run refetches while it is non-terminal and updates the verdict when it finishes, keeping the active tab.
- [ ] The "In progress" strip lists active runs with progress and disappears when none remain.
- [ ] Unmounting the page stops all timers.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and `references/data-and-state.md` before editing. Check whether the repository already has a poller component and reuse it if it fits.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-04-row-facts
- cqc-fe-06-run-drawer-header-and-checks
