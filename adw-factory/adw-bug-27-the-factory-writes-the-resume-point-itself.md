---
id: adw-bug-27-the-factory-writes-the-resume-point-itself
type: bug
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A hop depends on the agent writing a checkpoint, and agents often don't

> **Evidence, 2026-09-27.** Run `adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-…`
> (third attempt, after `adw-bug-26`): `build` reached the node cap (60) with
> **neither** `.adw/artifacts/build-checkpoint.md` nor `build.md` on disk, and
> the run blocked. The second attempt blocked the same way in `plan`: no
> checkpoint, though `plan.md` existed. By contrast, cost-04's build did write
> its checkpoint and hopped cleanly. Compliance with the "keep a checkpoint"
> instruction is intermittent. The worktree itself (the diff) is the ground
> truth of a build's progress, and it is always there.
> Constitution: *agents propose, code disposes*. The resume point must not
> depend on the agent.

## Requirements

- [x] **R1 — red first.** A `runCappedHops` test: the cap is reached, no
      checkpoint, no node artifact, and the workspace has uncommitted changes.
      The node must hop (fresh session, no `resume`) with a
      **factory-synthesized** resume block. This fails on `main`.
- [x] **R2 — synthesize from the worktree.** A pure renderer builds the block
      from deterministic inputs gathered at the edge: `git status --porcelain`
      (excluding `.adw/` and `.claude/`, the same `SWEEP_PATHSPEC` as commit),
      `git diff --stat`, and, when present, the upstream contract artifact
      (`plan.md` for build/test). Size-bound it (stat, not full diff). The hop
      preamble says it was synthesized by the factory because the session left
      no checkpoint, and tells the agent to re-derive progress from the diff,
      not restart.
- [x] **R3 — fallback order.** Checkpoint, then node artifact (adw-bug-26),
      then synthesized. Fail only when the workspace has no exec seam. An
      empty worktree still hops, with a block that says nothing was changed
      yet.
- [x] **R4 — journal it.** The hop records which source it used
      (`checkpoint | artifact | synthesized`) in its journal event, so a run
      shows how often agents skip the checkpoint.
- [x] **R5 — tests** for the synthesized path on both hop kinds (turn cap,
      poisoned session), and for the renderer's size bound.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then re-dispatch `adw-perf-06`.
