---
id: adw-m0-02-claude-md
type: chore
status: done
priority: 1
created: 2026-07-14
epic: adw-m0
depends: []
attempts: []
---
# CLAUDE.md — builder ground rules

## Context

Every agent session in this repo must inherit the constitution's two
non-negotiables without re-reading the full spec set.

## Deliverables

- `CLAUDE.md` at repo root, ≤ 30 lines.

## Requirements

- [x] Names `specs/constitution.md` as binding, and `specs/adw-v1.md` +
      `specs/adw-v1-plan.md` as source of truth for v1
- [x] States the TDD rule: no implementation code before a reviewed, red test
      (Art. I)
- [x] States the purity rule: pure functions with side effects at the edges,
      immutable structures, strict TS, errors carry `ticketId` + node name
      (Art. IX)
- [x] States the amendment rule: contradiction or gap found → stop and
      propose a spec amendment, never silently diverge
- [x] Points to `tickets/README.md` for the builder pickup protocol
      (24 lines total)

## Verify

File exists, ≤ 30 lines, contains all five points above.

## Out of scope

Duplicating spec content; per-directory CLAUDE.md files.
