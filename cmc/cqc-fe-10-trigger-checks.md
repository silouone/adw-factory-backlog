---
id: cqc-fe-10-trigger-checks
type: feat
status: queued
priority: 2
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-04-row-facts, cqc-fe-06-run-drawer-header-and-checks]
attempts: []
---
# Staff paste LO ids from Slack to trigger checks, and re-run a check from a row or the drawer

Triggering checks (spec `docs/cqc/spec-cqc-fe-release-1.md`, user stories
23–26; "API consumed": `GET /cqc/config`, `POST /cqc/checks`, `POST /cqc/batches`).
This is ship-plan step 2.

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
