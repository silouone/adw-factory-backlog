---
id: adw-graph-01-manual-tickets-keep-their-deps-c6db32
type: bug
status: in-progress
priority: 1
created: 2026-09-28
caps: {minutes: 60, turns: 250, stallMinutes: 20}
depends: []
attempts: []
---
# The backlog projection drops a manual ticket's `depends:`

> **Found 2026-09-28** while prototyping the dependency graph
> (`specs/adw-v1.14-backlog-tab.md` §10, 2026-09-28, **G10**).
>
> `cqc-be-11-deploy-dev-and-verify-…` (`type: manual`) declares 7 deps:
> `depends: [cqc-be-02-…, cqc-be-04-…, cqc-be-05-…, cqc-be-06-…, cqc-be-07-…,
> cqc-be-08-…, cqc-be-09-…]`. But `/backlog.json` serves it with `deps: []`.
> A graph then shows the release step as needing nothing.
>
> **Cause.** `buildLenientRow` in `src/web/backlog.ts` handles every file the
> strict `parseTicket` refuses, and a `manual` ticket is always refused
> (reserved operator type).
> - It hard-codes `deps: []` in `base`.
> - In the same function, the `epic` branch already reads `children:` with
>   `bracketedList(frontmatterField(frontmatter, …))`.
> - So manual rows lose their deps.
>
> Epics have no deps field, and malformed rows are out of scope. Leave both
> unchanged.

## Requirements

- [ ] **R1 — red first.** Add a `test/web/backlog.test.ts` case: a store holding
      a `type: manual` ticket with `depends: [a, b]`, where `a` is `done` and
      `b` is `queued`. `loadBacklog` must return that row with
      `deps: [{id:"a", status:"done", met:true, hard:false},
      {id:"b", status:"queued", met:false, hard:false}]`. Watch it fail on
      today's `deps: []`.
- [ ] **R2 — the fix.** A manual row's `deps` are resolved by **the same code
      path** as a parsed ticket's deps (the existing dep resolver above
      `deriveRowState`). Do not add a second copy of the met/hard rules.
      - An unknown dep id resolves the way the parsed-ticket path already
        resolves it.
      - `kind`, `status`, `closed` and every other field of the manual row are
        unchanged.
- [ ] **R3 — no other row kind changes.** Epic and malformed rows keep
      `deps: []`. The existing backlog tests stay green, unmodified.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then `just web` →
`/backlog.json`: `cqc-be-11-…` carries its 7 deps.
