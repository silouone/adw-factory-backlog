---
id: adw-console-01-operator-console
type: feat
status: rejected
priority: 2
created: 2026-09-14
depends: [adw-fe-12-wire-the-run-screen]
attempts: []
---
# The operator console — filters, dependency view, and a UI worth using

> **SUPERSEDED 2026-09-19 — `status: rejected`.** D1 was answered 2026-09-18
> (Preact); D3 is answered **yes** by `specs/adw-v1.14-backlog-tab.md`. Of
> this ticket's scope, "filter and group runs by target" shipped in
> `adw-fe-14`, and "dependency visibility across tickets" is the backlog tab
> (`adw-backlog-01`, `adw-backlog-02`). Nothing here is owed work.

> **BLOCKED ON APPROVAL — the status field says so, not just this banner.**
> `specs/adw-v1.5-operator-console.md` is *proposed*, not approved. It reverses
> v1.2 decision 11 ("zero new dependencies"), which is an operator call.
>
> This ticket carries `status: blocked` in its frontmatter deliberately.
> `adw-store-01-tickets-dir` wrote the same warning in prose while its status
> said `queued`, so `just next` listed it, it was dispatched, and a run spent
> ~15 minutes rediscovering what its own body said. A constraint the machine
> cannot read is not a constraint.

## Context

Full argument and the exact deltas in `specs/adw-v1.5-operator-console.md`.
In short: v1.2 bought observability and deliberately not a product. Having used
it, the operator wants the product.

## Blocked on — in order

1. **D1: does the factory take a frontend dependency?** Hand-written CSS, one
   small styling dependency, or a framework with a build step. Until this is
   answered nothing here is buildable.
2. `adw-fe-12-wire-the-run-screen` — there is no point restyling screens that
   are not yet reachable.
3. **D3 interacts with `adw-v1.4-ticket-store.md`** (also proposed): a
   dependency view means the web layer reads the ticket store, which v1.4
   proposes to move out of the repo. Settle or at least read v1.4 first.

## Scope once unblocked

- [ ] Filter and group runs by target. **No new data needed** —
      `run-start.target` has been journaled since `adw-fe-01`.
- [ ] Dependency visibility across tickets — the `depends:` graph, and what is
      blocked on what. Crosses into the ticket store (see D3).
- [ ] Navigation and styling to whatever D1 licenses.
- [ ] The console stays **read-only**. No mutating route, per v1.2's non-goals
      — a console that can dispatch is a different amendment.

## Explicitly deferred, with the reason

**Host RAM / CPU per run.** Not a view problem: nothing in the factory samples
host resources, so this means the engine sampling and journaling them per node
— new instrumentation in the hottest path for an unproven number. Measured
2026-09-14, a whole run's own processes are **~400 MB** against a machine
already carrying ~24 GB of editor and browser. The factory's footprint is close
to noise, and the operator's real memory question that day turned out to be
VS Code and Chrome, not the factory. Revisit when a concrete question needs it.
