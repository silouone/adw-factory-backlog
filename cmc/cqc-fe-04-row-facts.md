---
id: cqc-fe-04-row-facts
type: feat
status: queued
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [cqc-fe-03-list-of-checked-los]
attempts: []
---
# Each Content quality row tells staff how old, how stale and how connected the problem is

Row enrichment for the Content quality list (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 10–15; type shapes in the "Key type shapes" section).

## What to build

On every list row, from the `ContentRow` and `RunSummary` fields the service already returns:

- **Authoring tool**: Rise 360, Storyline, Captivate, iSpring, Adapt, Lectora
  or Unknown, plus embedded tools. `null` authoring shows `N/A`.
- **Connected or Static**: "Connected (served by <host>)" when `connected` is
  true, listing `content_hosts`; "Static" when false; `N/A` when null.
- **Timing facts**: first detected, last run, days open, run count, runner (E2B or local).
- **"Revision changed, not re-checked"** when `revision_changed` is true.
- **"Check failed to run"** with its `status_reason` when `last_failed_run` is
  newer than `latest_run`. It is never styled as Fail-block.
- **Queued or running** with its progress, when `active_run` is set.

## Acceptance criteria

- [ ] A page test with mocked rows shows each fact above, including `N/A` for each null field.
- [ ] A row with `last_failed_run` reads "Check failed to run" with its reason and does not carry the failed/Fail-block styling or wording.
- [ ] A row with `active_run` shows its lifecycle status and progress.
- [ ] A row with `revision_changed: true` shows "Revision changed, not re-checked".
- [ ] Every status is shown with an icon and a word, never colour alone.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and the conventions it routes to before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).
- No invented data.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-03-list-of-checked-los
