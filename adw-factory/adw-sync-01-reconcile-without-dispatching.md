---
id: adw-sync-01-reconcile-without-dispatching
type: feat
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-sync-01-reconcile-without-dispatching-1789372886529","branch":"adw/adw-sync-01-reconcile-without-dispatching","workspace":"/Users/silouane/adw-factory/runs/adw-sync-01-reconcile-without-dispatching-1789372886529/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The only way a ticket reaches `done` is to dispatch another ticket

> Minted 2026-09-14 after twelve hand-written ledger commits in one day.

## Evidence

`syncPrState` works. It queries gh for every `in-review` ticket with a recorded
PR and moves it: MERGED → `done`, CLOSED → `rejected`. That code is correct and
tested.

It has exactly one call site:

```
src/cli.ts:286   inside runSelected()   ← the DISPATCH path
```

`adw status`, `adw clean` and `adw web` never call it (`src/cli.ts:1473-1500`).
**So the ledger only advances as a side effect of spending tokens on something
else.** Stop dispatching and it freezes.

Measured cost on 2026-09-14: **12 commits of the shape
`tickets: X → done`**, all written by hand. And
`adw-auto-01-baseline-gate-snapshot` sat `in-review` for **two days** after
PR #11 merged, which is also why `just next` hid two runnable tickets behind a
dependency that was already satisfied.

That last part is the real damage. A stale ledger is not cosmetic once
`adw-depends-enforce` has landed: `depends:` is now enforced, so an unclosed
ticket **blocks** its dependents.

## Requirements

- [ ] Reconciling is reachable **without dispatching**. `adw status` is the
      obvious home (it already loads the target and walks `tickets/`), but a
      separate verb is equally acceptable — decide and say why.
- [ ] It must stay **safe to run at any time**, including with runs in flight:
      `commitTicketFile`'s pathspec scoping and the cross-run `_repo.lock`
      (`adw-par-01`) are what make that true today, and both must still hold.
- [ ] A read-only mode. An operator must be able to ask "what would this
      change?" without it changing anything — `status` that silently mutates
      the ledger would be a worse surprise than the current staleness.
- [ ] Each reconciliation stays journaled/reported as it is today (Art. VI) —
      this ticket changes WHEN the sweep runs, not what it reports.

## Verify

- With no dispatch at all, a ticket whose PR merged reaches `done`.
- Running it twice is a no-op the second time.
- Running it while a dispatch is in flight neither corrupts the ledger nor
  deadlocks against `_repo.lock`.
- `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Changing what `syncPrState` decides (MERGED → done, CLOSED → rejected, OPEN →
no write). Only its reachability is wrong.
