---
id: adw-fe-04-heartbeat
type: feat
status: done
priority: 1
created: 2026-09-12
depends: [adw-caps-01-ceiling-deterministic-tail, adw-fe-03-prompt-persistence]
attempts: [{"runId":"adw-fe-04-heartbeat-1789339599701","branch":"adw/adw-fe-04-heartbeat","workspace":"/Users/silouane/adw-factory/runs/adw-fe-04-heartbeat-1789339599701/workspace","outcome":"green","provider":"claude","model":"sonnet","pr":"24"}]
---
# Heartbeat — so a run that says `running` can be wrong

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

The factory cannot tell a dead run from a working one. A hard-killed lane
leaves a journal that simply *stops* — no `run-end`, no `abort`, no
`watchdog`, no `hard-stop`. The `adw-m5-06` run log records the cost in the
operator's own words: a run *"died hard… the ticket was left `in-progress`
with `attempts: []`. Reset to `queued` by hand."*

**Blocked by `adw-caps-01`: both edit `engine.ts`.**

## Requirements

- [ ] A periodic `heartbeat` journal event, armed on the engine's existing
      single-shot timer seam — so it is a self-rescheduling chain, not an
      interval. Cadence **15 s**, a named constant carrying its rationale.
- [ ] **`unref`'d**, matching the established precedent. A pending heartbeat
      must never by itself keep the lane alive — otherwise the mechanism meant
      to detect a wedged run is what makes it look healthy.
- [ ] **Bare payload.** The current node is already derivable from a
      `node-start` with no matching `node-end`; restating it is duplicated
      derivable state (Art. VIII).
- [ ] Engine-owned, like the journal middleware — a node cannot run
      un-heartbeated any more than it can run unjournaled.
- [ ] Cancelled on `run-end`, on abort, and on every terminal path.
- [ ] A named staleness constant (**60 s** — four missed beats) exported for
      readers. This ticket defines it; rendering it is `adw-fe-08`.

## Verify

- [ ] With a fake timer and fake clock: beats appear at cadence; the chain
      cancels on run-end, on abort, and on a thrown node; the timer is unref'd;
      **no beat is emitted after a terminal event**.
- [ ] A journal with no heartbeats at all (written before this change) still
      reads.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Pid recording and any kill capability — the view is read-only by operator
decision. A heartbeat proves the **process** is alive, not that it is making
progress; an alive-but-looping run stays undetectable in v1, and the spec says
so out loud.
