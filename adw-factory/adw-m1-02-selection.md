---
id: adw-m1-02-selection
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-01-ticket-contract]
attempts: []
---
# Ticket selection

## Context

"Select the named ticket, or else the highest-priority queued chore" (S1.1).
Selection is pure code over parsed tickets — no agent judgment (Art. III).

## Deliverables

- `test/intake/select.test.ts` (first, red)
- `src/intake/select.ts` — pure: `(tickets, namedId?) → Selection`

## Requirements

- [x] `--ticket <id>` names the winner; named id not found → descriptive
      error result (carrying the id) (S1.1)
- [x] Otherwise: highest priority (1 beats 3) among `status: queued` tickets
      whose `type` has a registered lane; tiebreak oldest `created`; final
      tiebreak lowest id (full determinism)
- [x] Non-queued and unregistered-type tickets are never auto-selected
      (satisfied by construction: Ticket.type is the literal 'chore'; parser
      rejects others — no runtime guard)
- [x] Empty queue → clean "nothing to do" result, not an error

## Build protocol (Art. I)

1. Tests: named hit / named miss; priority ordering; created tiebreak; id
   tiebreak; queued-only filter; empty queue.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

Refusing runs on in-progress/in-review tickets (run guard, adw-m2-05).
