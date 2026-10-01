---
id: hq-01-buildgraph-is-pure-and-tested-0aabb3
type: feat
status: in-progress
priority: 1
created: 2026-10-01
caps: {minutes: 120, turns: 300}
depends: [hq-00-bind-the-remote-2c5ede]
attempts: []
---
# The graph is a pure function of its inputs, and is tested

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

**Prefactor, which makes every later ticket easy.** Today the graph is built inside the server
module by reading the filesystem, ssh, `claude agents` and the factory directly. Extract
`buildGraph(inputs)`: a pure function over injected inputs (a home-folder reader, the M1
snapshot, the factory's work and runs, the `claude agents` output, launchd plists, the config).
The server becomes a thin edge that gathers inputs and calls it. **No behaviour change**: the
landing renders the same graph.

## Red first (Art. I-style, spec "Testing Decisions", seam 1)

Fixture inputs, then assert on the graph:
- clusters: first matching config regex wins; the catch-all takes the rest;
- skills merge by name across sources, with `portable`, `claudeOnly` and `codexOnly` set exactly as the spec defines;
- routine schedules: interval, calendar (with and without weekday), and always on;
- cluster counts: repos, memories, sessions, waiting, open, blocked, review, live;
- factory worktrees (`adw-wt/`, `runs/`) are not repos;
- memory machine marks (mbp, m1, both);
- the allowlist holds exactly the file-backed and output-backed ids.

## Acceptance criteria

- [ ] `buildGraph` performs no I/O (a test passes inputs only; no filesystem touched).
- [ ] The server's `/graph.json` for the real machine is unchanged in shape and counts (compare before and after).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-00-bind-the-remote-2c5ede
