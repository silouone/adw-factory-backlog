---
id: hq-11-an-m1-scan-survives-a-missing-cache-933bf9
type: bug
status: in-progress
priority: 2
created: 2026-10-02
depends: []
attempts: []
---
# A successful M1 scan shows up on a fresh clone, even before `cache/` exists

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Story 44. In `src/server.ts` `scanM1`, `writeFileSync(SNAP, …)` sits inside the same `try` as `JSON.parse`. Only `hq:install` creates `cache/`. On a fresh clone started with `bun start`, the write throws, and a **successful** scan is thrown away and reported as "offline · never scanned".

Fix the seam, not just the folder: the scan result is decided by the parse, and a write failure is reported, never silently treated as an offline M1.

## Red first

- Extract the decision into a pure function, or inject the writer, so it can be tested. With an injected writer that throws, a valid scan stdout still gives `state: "live snapshot"` and the snapshot is kept.
- Invalid stdout still falls back to the last snapshot, with its age (behaviour unchanged).

## Acceptance criteria

- [ ] The server creates `cache/` at startup if it is missing (`mkdir -p` semantics).
- [ ] A write failure is logged with the path. It never downgrades a live scan.
- [ ] The 60 s ssh kill is unchanged.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- (nothing)
