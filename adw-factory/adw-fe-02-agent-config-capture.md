---
id: adw-fe-02-agent-config-capture
type: feat
status: done
priority: 1
created: 2026-09-12
depends: [adw-fe-01-journal-schema]
attempts: []
---
# Retain the SDK `system/init` message instead of discarding it

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

`build.ts` reduces the SDK's `system/init` message to `session_id` and drops
everything else **one line into the consumer**. That message already carries,
on the wire, today: resolved model, tool list, MCP servers + status, skills,
plugins, slash commands, permission mode, CLI version.

This is the highest value-per-line change in the whole v1.2 spec: the data
arrives free and is thrown away.

## Requirements

- [ ] Widen the SDK message mirror so `system/init`'s fields survive. The
      mirror is deliberately narrow — widen it deliberately too, naming the
      fields kept rather than passing the message through wholesale.
- [ ] Journal them **inline** on the agent node — this payload is small.
- [ ] Absent or partial fields are tolerated: an older CLI that omits one must
      not fail the run (Art. VI — a sick sink never blocks a green run).

## Verify

- [ ] With a fake query seam emitting a full `system/init`, every named field
      reaches the journal.
- [ ] A minimal `system/init` (session_id only) journals what exists and does
      not throw.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

The system prompt — it is **not** in the SDK message union at all and cannot
be recovered here (see `adw-sysprompt-01`).
