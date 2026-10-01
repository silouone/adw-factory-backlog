---
id: adw-web-02-a-one-shot-board-snapshot-75da7d
type: feat
status: blocked
priority: 3
created: 2026-10-01
depends: []
attempts: [{"runId":"adw-web-02-a-one-shot-board-snapshot-75da7d-1790883174675","branch":"adw/adw-web-02-a-one-shot-board-snapshot-75da7d","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-web-02-a-one-shot-board-snapshot-75da7d-1790883174675/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# adw web serves the board as a one-shot JSON snapshot

A consumer that wants the board once (the HQ's factory widget, spec `~/personal_project/silou-hq/docs/spec-v1.md`) has to open the
`/events` SSE stream, read its first frame and hang up. A plain `GET /board.json` returning
the same snapshot (`views`, without the run-screen-only payloads) is simpler and cacheable.

## What to build

- `GET /board.json`: the same view-model snapshot `/events` sends first, token-checked, read-only.

## Acceptance criteria

- [ ] `/board.json` equals the first `/events` frame's `views` for the same state (test).
- [ ] The no-mutating-route guard still passes.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
