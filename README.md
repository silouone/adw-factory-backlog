# ADW backlog

The central ticket store for the ADW factory (`~/personal_project/adw-factory`),
per `specs/adw-v1.4-ticket-store.md` §10. There is one subdirectory per target,
and each target's `ticketsDir` points at its own subdirectory. The factory commits
ticket status transitions here, so a client repository is never written to by a run.

- `cmc/`: Go1 Content Management Console (CQC frontend release 1)
- `sabado/`: Sabado (`~/personal_project/SABADO/sabado`), moved from the repo's `tickets/` on 2026-09-27
- `clens/`: cLens (`~/agent-observability-project`), moved from the repo's `tickets/` on 2026-09-27
