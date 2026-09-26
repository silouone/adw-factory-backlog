---
id: cqc-fe-12-resolve-validate-reopen
type: feat
status: queued
priority: 3
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [cqc-fe-05-summary-tabs-search-filters, cqc-fe-06-run-drawer-header-and-checks]
attempts: []
---
# Content ops resolve, validate and reopen an LO with an outcome and a note

The case workflow for release 1 (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 50–54; "API consumed": `POST` and `DELETE /cqc/content/{lo}/resolution`).
This is ship-plan step 3. The case workflow beyond resolve and validate is out of scope.

## What to build

- **Service:** `POST /cqc/content/{lo}/resolution` (outcome and note) and
  `DELETE /cqc/content/{lo}/resolution` (reopen).
- **Resolve…** on open LOs, on the row and in the drawer footer. It opens a
  modal owned by its trigger, with:
  - an outcome: fixed and verified, validated manually, false positive, or accepted as is
    (contract values `fixed_verified`, `validated_manually`, `false_positive`, `accepted_risk`);
  - a required note of at least 10 characters (the contract rejects shorter notes).
- **"Fixed and verified"** is selectable only when the LO's latest run passes.
  Otherwise it is disabled with the reason.
- **Validate** on passing LOs records a human confirmation through the same
  endpoint and modal, with the outcome `validated_manually` preselected.
- A `409 latest_run_not_pass` response (a newer run no longer passes) shows
  its reason in the modal and refetches the LO.
- **Resolution display:** who resolved it, when, the outcome and the note, in
  the drawer case line and on the Resolved tab.
- **Automatic reopen:** when `resolution.active` is false with
  `reopened_by_run_id`, the drawer shows the reopened banner linking to that run.
- **Reopen** on a resolved LO calls `DELETE` and moves it back to Open.
- On success: a confirmation, and the list and tab counts refetch.

## Acceptance criteria

- [ ] Submitting with an empty note or a note under 10 characters shows a validation error and sends nothing.
- [ ] A mocked `409 latest_run_not_pass` shows its reason in the modal and does not close it.
- [ ] "Fixed and verified" is disabled with its reason when the latest run is not Pass, and enabled when it is.
- [ ] A valid submit calls the resolution endpoint with the outcome and note, shows a success confirmation, closes the modal, returns focus to its trigger and refetches the list.
- [ ] A resolved LO shows who, when, the outcome and the note, and offers Reopen. Reopen calls `DELETE` and the LO returns to Open.
- [ ] Validate appears only on passing LOs.
- [ ] A reopened resolution shows the reopened banner with a link to the reopening run.
- [ ] Service tests assert exact paths and bodies.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md`, the modal recipe, `rules/react/modal-trigger-ownership.md` and `references/testing.md` before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`.
- Go1d: verify components against the installed version (see cqc-fe-03 for the stated-exception path).

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

- cqc-fe-05-summary-tabs-search-filters
- cqc-fe-06-run-drawer-header-and-checks
