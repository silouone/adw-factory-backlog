# ADW backlog

The central ticket store for the ADW factory (`~/personal_project/adw-factory`),
per `specs/adw-v1.4-ticket-store.md` §10. There is one subdirectory per target,
and each target's `ticketsDir` points at its own subdirectory. The factory commits
ticket status transitions here, so a client repository is never written to by a run.

- `cmc/`: Go1 Content Management Console (CQC frontend release 1)
- `cqc/`: Content Quality Checker backend (`~/work_project/coorp/content-quality-checker`, target `content-quality-checker`), release 1 read API
- `sabado/`: Sabado (`~/personal_project/SABADO/sabado`), moved from the repo's `tickets/` on 2026-09-27
- `clens/`: cLens (`~/agent-observability-project`), moved from the repo's `tickets/` on 2026-09-27
- `adw-factory/`: the factory's own meta-backlog, moved from `tickets/` on 2026-09-27 (adw-store-02); `tickets/README.md` (pickup protocol) and `tickets/BACKLOG.md` stay in the code repo
