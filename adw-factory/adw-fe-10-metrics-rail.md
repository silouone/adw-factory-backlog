---
id: adw-fe-10-metrics-rail
type: feat
status: done
priority: 2
created: 2026-09-12
depends: [adw-fe-06-gantt, adw-fe-02-agent-config-capture, adw-fe-07-tool-dots]
caps: {minutes: 120, turns: 600}
attempts: [{"runId":"adw-fe-10-metrics-rail-1789338922316","branch":"adw/adw-fe-10-metrics-rail","workspace":"/Users/silouane/adw-factory/runs/adw-fe-10-metrics-rail-1789338922316/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/21","provider":"claude","model":"sonnet"}]
---
# The metrics rail — model, context, and the rates that diagnose a run

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

The argument for this view beyond liveness. On the `adw-par-01` run,
seconds/turn read `plan 18.1s · build 6.6s · test 6.9s` and the tool/model
split showed `plan` was **99% model, 1% tool** (33 Bash calls averaging 0.2 s).
That pair distinguishes "the model is generating a long document" from "the
test suite is slow" from "the agent is thrashing" — and **no existing surface
can answer it**.

## Requirements

- [ ] Per-role left rail carrying the resolved **model**.
- [ ] Per-node derived metrics, none stored: duration, turns, tokens,
      **seconds/turn**, **tokens/second**, and the **tool-time vs model-time**
      split (Σ tool `duration_ms` vs duration minus that).
- [ ] **`tokens` is labelled as an undercount** — "billed in/out", never
      "tokens used". It excludes cache reads, which on a cached run are most of
      the real prompt.
- [ ] **Blocked-on-subagent time renders as its own band**, not as model time.
      The same run spent **12.1 of the test node's 23 minutes** inside blocking
      subagent waits, and because subagent turns are excluded from the budget
      that time was invisible in every existing metric — it never appeared in
      the 422-turn count that killed the run.
- [ ] **Context occupancy**: numerator is the last valid assistant turn's
      `input + output + cache_read + cache_write` (cached prompt is still
      prompt). If a model→window map proves contentious, **ship an absolute
      token count rather than a percentage** — a real number beats a percentage
      of a guessed denominator.
- [ ] A run predating the breakdown renders what it has and says what is
      missing, rather than showing zeroes.

## Verify

- [ ] Every metric is asserted at the projection from fixture records.
- [ ] A pre-`adw-fe-01` journal renders without claiming a model or a rate.
- [ ] Subagent-blocked time is attributed to its own band.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
