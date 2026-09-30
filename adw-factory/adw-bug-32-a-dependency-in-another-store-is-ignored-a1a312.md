---
id: adw-bug-32-a-dependency-in-another-store-is-ignored-a1a312
type: bug
status: queued
priority: 1
created: 2026-09-30
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A dependency on a ticket in another target's store is silently treated as met

Measured 2026-09-30. The release-3 CMC tickets depend on CQC backend tickets:
`cqc-fe-22` needs `cqc-be-28`, and `cqc-fe-26` needs `cqc-be-34`. Their authors could only
write those blockers as **text**, because `depends:` can't express them. Both tickets were
dispatched while their backend dependency was in review or still queued. Both agents
correctly refused to build, reading the blocker from the ticket body. The runs then fell
into gates and repair anyway (see adw-bug-31 and adw-bug-33) and cost $3.30.

## Why

`unmetDependsRefusal` (`src/cli.ts:686-711`) looks each dependency up in `allTickets`, which
holds **one target's store** (`cli.ts:502`). A dependency id it doesn't find is
"dangling" and is **treated as met** on purpose (`cli.ts:690-693`). So a cross-store id in
`depends:` doesn't block anything. It silently passes.

Ticket ids are unique across **all** stores by design (spec `adw-v1.4-ticket-store.md`
§10.2, the uid suffix), so a global lookup is unambiguous.

## Red first (Art. I)

- `unmetDependsRefusal(cmcTicket{depends:[cqcId]}, cmcTickets)`, with the CQC ticket queued
  in the **CQC** store, must refuse. Today it returns `undefined`.
- A dependency id that exists in **no** store must refuse, naming it as unknown. Today it
  counts as met.

## Acceptance criteria

- [ ] Dependencies resolve across every store under the backlog root (every target's
      `ticketsDir`). An unmet cross-store dependency refuses dispatch, naming its target.
- [ ] A dependency id found in no store refuses dispatch, and `just tickets` / `just next`
      report it. This is adw-depends-enforce's deferred requirement 2; close it here.
- [ ] `just next` never offers a ticket with an unmet cross-store dependency.
- [ ] The backlog projection (`src/web/backlog.ts`) and graph show cross-store edges.
- [ ] Spec amendment first (Amendment rule): add cross-store `depends:` to
      `adw-v1.4-ticket-store.md`.
- [ ] Follow-up for the operator, not code: move the text blockers on cqc-fe-22..26 into
      `depends:`.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
