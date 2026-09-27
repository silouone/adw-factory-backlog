---
id: adw-m8-01-ticket-type-union
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m1-01-ticket-contract]
attempts: []
---
# Widen the ticket type union to `chore | bug | feat`

> Source of truth: `specs/adw-v1.1-lanes.md` (decision 2). Read the constitution
> + `adw-v1.md` + `adw-v1-plan.md` first (protocol §1).

## Context (grounded in source)

- `src/intake/ticket.ts:57` — `Ticket.type` is the literal `"chore"`.
- `src/intake/ticket.ts:77` — `REGISTERED_LANES: readonly string[] = ["chore"]`;
  parse rejects a `type` with no registered lane (`ticket.ts:129`, S1.3).
- `src/intake/ticket.ts:210` — the built `Ticket` hardcodes `type: "chore"`.
- Tests: `test/intake/ticket.test.ts` (pure, exhaustive validation cases).

## Requirements

- [ ] `Ticket.type` union widened to `"chore" | "bug" | "feat"`.
- [ ] `REGISTERED_LANES` = `["chore", "bug", "feat"]`.
- [ ] The parsed `Ticket` carries the ticket's **actual** validated type, not a
      hardcoded `"chore"`.
- [ ] A ticket whose `type` is none of the three is still **rejected** with the
      existing `unknown type "<x>": no registered lane` error (S1.3 unchanged).
- [ ] No other field, node, or contract changes (this ticket is the union only).

## Build protocol (red-first, Art. I / protocol §4)

1. Extend `test/intake/ticket.test.ts`: `bug` and `feat` parse to a `Ticket`
   with the matching `type`; a fourth type (e.g. `hotfix`) is rejected. Confirm
   red, present for review.
2. Implement the union + registry + actual-type wiring to green.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; the new type cases pass;
every pre-existing intake test stays green (no regression to chore parsing).

## Out of scope

Templates, lanes, the bug graph — later children.
