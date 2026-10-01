---
id: hq-03-the-factory-adapter-is-a-contract-a06397
type: feat
status: queued
priority: 1
created: 2026-10-01
depends: [hq-01-buildgraph-is-pure-and-tested-0aabb3]
attempts: []
---
# The factory is read through one adapter, pinned by contract tests

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Move the factory read into its own module: given the env URL, fetch `/backlog.json` and the
first `/events` frame; return `{reachable, work, live, recent}`. That is the only place the
factory's shapes appear. If adw-factory ships `/board.json` (adw-web-02), prefer it, and fall
back to the SSE frame.

## Red first (fixtures of the real payloads)

- `/backlog.json` rows map to open work (closed, done, rejected and epic rows excluded), and projects map to clusters through `config.factory.projectClusters`, falling back to the cluster regexes.
- Run views split into live (`state: running`) and recent (`finished`, within 72h, newest first, capped). The run time is derived from the `<ticketId>-<epochMs>` runId.
- Unreachable (timeout, HTTP error, malformed frame) returns `reachable: false` and never throws.
- Not configured returns `reachable: false, configured: false`.

## Acceptance criteria

- [ ] No other module references factory payload fields.
- [ ] With adw web stopped, the landing still loads and the factory widget says it is unreachable.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-01-buildgraph-is-pure-and-tested-0aabb3
