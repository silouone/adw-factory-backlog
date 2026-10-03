---
id: adw-gates-12-sabado-runs-the-size-ratchet-before-it-pushes-3b3b0d
type: chore
status: in-progress
priority: 1
created: 2026-10-03
caps: {minutes: 45, turns: 120}
depends: []
attempts: []
---
# sabado runs the size ratchet as a gate, so a factory PR stops poisoning main

## Why

sabado's CI checks a repo-wide size ratchet: `scripts/size-ratchet.py`. No tracked backend or
frontend source file may grow past 1 000 lines or past its `size-baseline.json` entry.
`targets/sabado.json` has no gate for it. Every factory run against sabado can therefore pass all
its local gates and still push a file over the line.

That is exactly what happened. Four backend-only factory PRs merged growth that no gate checked:

| PR | Branch | Grew |
|---|---|---|
| #1333, #1334, #1336 | adw/sabado-42, -37, -38 | `chat_loop.py` 1058, `ai_tools.py` 1004, `ai_tools_writes.py` 1002 |
| #1355 | adw/sabado-46 | `extraction/rate_limit.py` 1226 → 1227 |

The red then landed on a human contributor's next six front PRs, none of which had grown
anything. Since 2026-10-02, about 15 of 18 red sabado PR runs were not the PR's own fault. The
full audit is in `ai_docs/2026-10-03-sabado-red-ci-audit.md`.

sabado's own CI fix is #1351: the ratchet becomes its own job, run on every PR and every push.
It closes the hole on the GitHub side. This ticket closes it on the factory side, so the factory
never opens a PR that CI will reject for growth.

## What to build

- [ ] **R1** Add a gate to `targets/sabado.json`:
      `{ "name": "size-ratchet", "cmd": "python3 scripts/size-ratchet.py" }`. It runs from the
      repo root, because the script checks both trees. It is stdlib-only and needs no venv.
      Place it **first**: it takes about 1 s and is the cheapest gate to fail.
- [ ] **R2** The target loader still accepts the config (`src/targets/loader.ts`). Add or extend
      a test that loads `targets/sabado.json` and asserts the `size-ratchet` gate is present, so a
      later edit cannot silently drop it.
- [ ] **R3** Do not change any other gate, and do not touch sabado's repo.

## Out of scope

- A generic "target gates must mirror CI" check. That is worth a spec note, not this ticket.
- Splitting the god files: `sabado-62`.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test` is green.
- [ ] In a sabado checkout on current main, `python3 scripts/size-ratchet.py` exits 0. After
      appending one line to `backend/app/extraction/rate_limit.py`, it exits 1 and names the file.
      This is the gate's red/green proof, so record both outputs in the PR body.
