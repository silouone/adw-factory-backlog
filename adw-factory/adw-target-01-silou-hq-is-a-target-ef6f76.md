---
id: adw-target-01-silou-hq-is-a-target-ef6f76
type: chore
status: in-progress
priority: 1
created: 2026-10-01
depends: []
attempts: []
---
# silou-hq is a factory target

The HQ (`~/personal_project/silou-hq`, spec `~/personal_project/silou-hq/docs/spec-v1.md`) is built by this factory. Today
`TARGET=silou-hq just next` refuses, because no target config exists.

## Deliverables

- A `silou-hq` target config, modelled on the `adw-factory` target:
  - repo `/Users/silouane/personal_project/silou-hq`, base `main`, branch prefix `adw/`;
  - tickets in `~/adw/backlog/silou-hq`;
  - gates `lint` (`bun run lint`), `typecheck` (`bunx tsc --noEmit`) and `test` (`bun test` with a junit report to `.adw/report.xml`, declared as its report);
  - setup `bun install`;
  - context `CLAUDE.md` and `docs/spec-v1.md`.
- One line for `silou-hq/` in the store's `README.md`.

## Acceptance criteria

- [ ] The target config parses: `just targets` lists `silou-hq`.
- [ ] `TARGET=silou-hq just next` lists the silou-hq store's queued tickets whose dependencies are met.
- [ ] The `test` gate's junit report is produced in a silou-hq checkout (`bun test --reporter=junit --reporter-outfile=.adw/report.xml`).
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green in adw-factory.

## Blocked by

- (nothing)
