---
id: adw-m1-01-ticket-contract
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m0-01-toolchain]
attempts: []
---
# Ticket contract — parse + validate

## Context

The ticket file is the only ticket state (Art. VIII). Everything downstream
consumes the typed structure this module produces. Contract: plan §5
"Ticket". Rejection must be deterministic and free — no workspace, no agent
tokens (S1.3).

## Deliverables

- `test/intake/ticket.test.ts` (first, red)
- `src/intake/ticket.ts` — pure: `(fileName, rawContent) → ParseResult`

## Requirements

- [x] Parse YAML frontmatter + markdown body of `tickets/<id>.md` into an
      immutable `Ticket` (S1.2)
- [x] Required fields: `id` (non-empty, must equal filename stem), `type`
      (must have a registered lane — v1: `chore` only), `status`
      (`queued|in-progress|in-review|done|blocked|rejected`), `priority`
      (1|2|3), `created` (ISO date). Optional: `model` (string), `caps`
      (`{minutes: positive int, turns: positive int}`), `attempts`
      (array, defaults `[]`) (plan §5)
- [x] Invalid input → `{ok: false, errors}` where `errors` names **every**
      missing/invalid field with field name + reason, in one deterministic
      pass (S1.3); valid → `{ok: true, ticket}` (discriminated union)
- [x] Pure function: no I/O, no clock, no randomness; strict TS throughout
      (Art. IX)

## Build protocol (Art. I)

1. Tests covering: minimal valid ticket; valid with `model`/`caps` overrides;
   each required field missing; unknown `type`; bad `status`/`priority`/
   `caps`; id/filename mismatch; **multiple errors reported together**.
2. Confirm red: `bun test test/intake/ticket.test.ts`
3. Implement; green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

File reading (caller's job), selection (m1-02), status writes (m1-03).
