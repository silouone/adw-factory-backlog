---
id: adw-bug-26-a-hop-without-a-checkpoint-resumes-from-the-artifact
type: bug
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A turn-cap hop with no checkpoint file blocks the run, even when the node's own artifact is on disk

> **Evidence, 2026-09-27, run `adw-perf-06-bug-lane-and-fix-loops-hand-off-through-artifacts-1790480156025`.**
> `plan` reached the node cap (60 API calls, after `adw-bug-25`) and the run blocked:
> `turn cap (60) reached for node "plan" but no checkpoint artifact at .adw/artifacts/plan-checkpoint.md`.
> The session had written `.adw/artifacts/plan.md`, its own deliverable, **four
> times**, and never the checkpoint. The instruction block ("Keep
> `.adw/artifacts/plan-checkpoint.md` up to date…") was in the prompt. The
> artifact the node exists to produce is the better resume point, and it was
> ignored.
>
> Watch item (not in scope): that plan session's context reached **254k** at
> 60 calls (~4k/call). A call-count cap does not bound context for a
> read-heavy node.

## Requirements

- [x] **R1 — red first.** A `runCappedHops` test: the cap is reached, no
      checkpoint file exists, and the node's artifact (`plan.md` for `plan`) is
      non-empty. The node must hop (fresh session, no `resume`), and the new
      prompt must carry the artifact under a heading that names it as the prior
      session's output. This fails on `main`.
- [x] **R2 — fallback order.** Checkpoint first, then the node's own
      artifact (`artifactPath(name)`), then fail as today. The fail reason
      names **both** missing paths. The same fallback applies to the
      poisoned-session hop (`adw-bug-22`).
- [x] **R3 — tell the agent.** The hop preamble says which source it
      resumed from. For the artifact source, it tells the agent to treat the
      artifact as a draft to complete and correct, not to restart.
- [x] **R4 — tests for both hop kinds** (turn cap, poisoned session), and
      for both absent: the reason names both paths.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then re-dispatch `adw-perf-06`.
