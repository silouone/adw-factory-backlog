---
id: adw-depends-enforce
type: bug
status: done
priority: 1
created: 2026-09-12
depends: []
attempts: [{"runId":"adw-depends-enforce-1789340968553","branch":"adw/adw-depends-enforce","workspace":"/Users/silouane/adw-factory/runs/adw-depends-enforce-1789340968553/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/27","provider":"claude","model":"sonnet"}]
---
# `depends:` is documented, written on 30+ tickets, and enforced nowhere

## Symptom

    $ grep -rno "depends" src/ | wc -l
    0

The field appears **nowhere** in `src/`. `parseTicket` does not read it, no
`Ticket` carries it, and neither the pinned nor the unpinned dispatch path
consults it. `tickets/README.md` documents it ("ticket ids that must be `done`
first"), `cookbooks/run_ticket.md` step 0 tells the OPERATOR to check it by
hand, and 30+ tickets declare edges that the factory cannot see.

Found 2026-09-12 while splitting `specs/adw-v1.3-review-lane.md` into the M9
graph: six tickets with declared blocking edges, all six offered as immediately
runnable. `just run adw-m9-06-lane-wiring` would dispatch happily with none of
its five predecessors built.

## Why it matters

A dependency graph nobody enforces is worse than no graph: it reads as a
guarantee. The failure is also expensive and confusing rather than loud — the
agent gets a ticket whose prerequisites do not exist, and either flounders,
invents the missing seam, or blocks after burning a plan and a build. Three
runs today cost 30–90 minutes each; a dep-order mistake costs the same.

`selectTicket` (S1.1) picks "the lowest-priority-number ticket whose depends are
all done" per `tickets/README.md`. It does not, because it cannot — the field
never reaches it.

## Requirements

- [ ] `parseTicket` retains `depends` on the `Ticket` as a readonly string
      array. Absent → empty array, NOT undefined (the `attempts` precedent).
- [ ] Validation: every id in `depends` must name a ticket file that EXISTS. A
      dangling edge is a malformed ticket, reported pre-engine with the others
      in one pass. (Today a typo'd dep is silently ignored — my own
      `adw-m5-07` reference was dangling for an hour before I caught it by
      reading.)
- [ ] A ticket must not depend on itself, directly or transitively. A cycle is
      malformed, and the reported reason must name the cycle.
- [ ] `selectTicket` skips any ticket with an unmet dep, so the UNPINNED path
      finally matches its documented contract.
- [ ] The PINNED path (`--ticket <stem>`) REFUSES a ticket with unmet deps
      pre-engine, zero tokens, naming which deps are unmet and their statuses.
      Pinning is an override of *selection*, not of the dependency graph.
- [ ] An escape hatch for the operator to dispatch anyway — deliberate, named,
      and journaled. Sometimes a dep is done in substance but not in ledger
      (today: `adw-m5-06` merged while its ticket still read `in-review`).
      Propose `--ignore-depends`; the operator decides the spelling.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first: unmet dep refused on the pinned path with the dep named;
      met deps dispatch; a dangling dep id reported as malformed; a 2-cycle and
      a 3-cycle both reported with the cycle named; `selectTicket` skipping an
      unmet-dep ticket in favour of a lower-priority runnable one; the escape
      hatch dispatching and journaling that it was used.
- [ ] Verified against the REAL ledger: `adw-m9-06-lane-wiring` is refused
      today, and `adw-m9-01-review-verdict-contract` is not.

## Out of scope

`just next` already reads `depends` as of this commit, so the operator has a
guard — but a justfile recipe is not enforcement and the CLI must not rely on
it. Re-typing the legacy `type: feature` tickets. Auto-running a dep chain —
the operator remains the scheduler (plan §2 decision 11).
