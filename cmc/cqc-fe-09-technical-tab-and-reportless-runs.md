---
id: cqc-fe-09-technical-tab-and-reportless-runs
type: feat
status: queued
priority: 3
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [cqc-fe-07-drawer-navigation-runs-sharing]
attempts: []
---
# Engineers debug a run from its Technical tab, even when the run produced no report

Provenance and diagnostics in the run drawer (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 44, 45, 47).

## What to build

- **Technical tab** showing:
  - launch, profile, dependency mode, agent;
  - cost, duration;
  - runner, with an uncommitted-changes flag (`git_dirty`);
  - code version, prompt hash;
  - extraction and navigation stats;
  - the raw report and the metadata JSON.
- **"Not recorded for this run":** the fields a run did not record are listed on
  one line, instead of a grid of `N/A` tiles.
- **Runs without a report:** the drawer still shows its Runs tab (when the LO
  has several runs) and its Technical tab, so staff can diagnose why it
  produced nothing. Checks and Evidence explain that there is no report.

## Acceptance criteria

- [ ] With a full fixture, every Technical field renders, and the raw report and metadata JSON are viewable.
- [ ] With a fixture missing several fields, they appear together on the single "Not recorded for this run" line and nowhere as `N/A` tiles.
- [ ] A run with `report: null` renders the Technical tab and, for a multi-run LO, the Runs tab. The other tabs say no report was produced.
- [ ] `git_dirty: true` shows the uncommitted-changes flag with a word, not colour alone.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and the conventions it routes to before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-07-drawer-navigation-runs-sharing
