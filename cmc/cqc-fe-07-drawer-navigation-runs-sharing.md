---
id: cqc-fe-07-drawer-navigation-runs-sharing
type: feat
status: queued
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
- Read `AGENTS.md` and the conventions it routes to before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-06-run-drawer-header-and-checks
