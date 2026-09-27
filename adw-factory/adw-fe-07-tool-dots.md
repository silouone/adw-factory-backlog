---
id: adw-fe-07-tool-dots
type: feat
status: done
priority: 2
created: 2026-09-12
depends: [adw-fe-06-gantt]
attempts: []
---
# Tool-call density inside each node block

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

The per-lane ticks. This data lives only in the cLens capture JSONL — but that
file is **in our own run directory**, in a flat four-key shape, so reading it
is not reaching into another tool's internals.

Two hazards, both already paid for once:

**Session id is 1:N, not 1:1.** `repair` and `ci-repair` **resume** the
original session, so their tool calls append to the same capture file.
Attributing dots by session id alone renders repair activity as *builder*
activity.

**The capture directory is not trustworthy as a listing.** It is resolved by a
filesystem walk-up, so **any process whose cwd is inside the workspace captures
into it** — including the operator's own shell. On 2026-09-12 an operator
`bun test` landed a 405-second command in a run's capture dir and was briefly
attributed to the agent.

## Requirements

- [ ] Dots bind to a lane by **`timestamp ∩ node window`** (node-start →
      node-end), **never by session id alone**.
- [ ] Capture sessions are addressed **by the ids the journal names** in its
      `usage` / `capture` events. Never by globbing the directory; anything
      else in there is foreign and ignored.
- [ ] **"No capture recorded" renders distinctly from "captured, zero tool
      calls."** These are different facts — 18 of 69 sessions have the former —
      and conflating them is the same class of lie as `unknown`-as-`running`.

## Verify

- [ ] **A resumed-session tool call lands in the repair lane, not the builder
      lane.** The regression test for the 1:N hazard, and the single most
      valuable test in this ticket.
- [ ] A foreign session file in the capture dir is ignored.
- [ ] Missing capture and empty capture produce different views.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
