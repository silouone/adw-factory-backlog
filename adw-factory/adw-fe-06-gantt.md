---
id: adw-fe-06-gantt
type: feat
status: done
priority: 2
created: 2026-09-12
depends: [adw-fe-05-web-grid]
caps: {minutes: 120, turns: 600}
attempts: [{"runId":"adw-fe-06-gantt-1789315526125","branch":"adw/adw-fe-06-gantt","workspace":"/Users/silouane/adw-factory/runs/adw-fe-06-gantt-1789315526125/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/16","provider":"claude","model":"sonnet"}]
---
# The per-run Gantt — roles as rows, time as the x-axis

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

The shape the operator asked for, and the one thing cLens does **not** have:
its `TimelineTab` is a single-track ribbon and its only lane component runs
agents as *columns* with time flowing *down*. This is a Gantt — rows are roles,
x is time, blocks are sized by real duration.

## Requirements

- [ ] Lane assignment derives from what is already journaled: the `node-end`
      detail discriminates agent nodes from gate nodes from plain deterministic
      ones, and **for agent nodes the node name IS the role** (`plan` →
      planner, `build` → builder, …). Deterministic nodes share one `workspace`
      lane; gate results render as chips within it.
- [ ] Node blocks positioned and sized by real duration against a time axis.
- [ ] A node with no `node-end` renders as in-progress, not as zero-length.
- [ ] Retried nodes are legible as retries (from the `round` event).
- [ ] Navigable from a grid card and back.

## Verify

- [ ] Each node kind lands in its expected lane, asserted at the projection.
- [ ] An unfinished node renders in-progress.
- [ ] A run with a `round` event shows the retry.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
