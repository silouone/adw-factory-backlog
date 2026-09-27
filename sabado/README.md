# sabado tickets (ADW factory)

Work orders for the ADW factory (`~/personal_project/adw-factory`, target
`sabado`, `ticketsDir: ~/adw/backlog/sabado`). Moved here from
`<sabado repo>/tickets/` on 2026-09-27; the factory rewrites `status:` and
`attempts:` itself and commits those transitions in this store, never in the
sabado repository. The pickup protocol and the findings stay in the sabado
repo (`tickets/README.md`, `tickets/findings/`).

```bash
cd ~/personal_project/adw-factory
TARGET=sabado just next
TARGET=sabado just run sabado-25-the-built-front-boots-in-a-browser-before-it-ships
```
