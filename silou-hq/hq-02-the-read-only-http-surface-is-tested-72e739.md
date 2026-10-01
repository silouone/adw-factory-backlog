---
id: hq-02-the-read-only-http-surface-is-tested-72e739
type: feat
status: done
priority: 1
created: 2026-10-01
depends: [hq-01-buildgraph-is-pure-and-tested-0aabb3]
attempts: [{"runId":"hq-02-the-read-only-http-surface-is-tested-72e739-1790896749213","branch":"adw/hq-02-the-read-only-http-surface-is-tested-72e739","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-02-the-read-only-http-surface-is-tested-72e739-1790896749213/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/2","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The read-only HTTP surface is proven by tests

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Make the server constructible with injected dependencies, `serve(deps)`, and pin its safety
guarantees (spec "Testing Decisions", seam 2).

## Red first

- Every non-GET method on every path gets **405**.
- `/file` and `/output` with an id not in the allowlist, or a path-shaped id (`../`, `/etc/passwd`, `~`), get **404**.
- Tails are bounded (400 lines for files, 120 for outputs) even for a huge fixture file.
- `/factory.json` distinguishes **not configured** (no env var) from **not reachable** (configured, adapter failed).
- A source guard: the server module branches on no HTTP method except to refuse it (prior art: adw-factory's grep-tested no-mutating-route test).

## Acceptance criteria

- [ ] All of the above as tests against `serve(deps)` with fixture readers; no network.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-01-buildgraph-is-pure-and-tested-0aabb3
