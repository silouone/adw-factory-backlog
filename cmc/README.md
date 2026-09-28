# CMC tickets (ADW factory)

Work orders for the ADW factory (`~/personal_project/adw-factory`, target
`cmc`, `ticketsDir: ~/adw/backlog/cmc`). They live in the central backlog, never
in the Go1 repository. It rewrites `status:` and
`attempts:` itself; never edit `attempts:` by hand.

## Content quality page, release 1 (`cqc-fe-*`)

**Base branch: `cqc/release-1`** (operator decision 2026-09-28). The CLB team owns the CMC and
reviews from another timezone, so the page is built on a feature branch: `targets/cmc.json`
has `"base": "cqc/release-1"`, every factory PR targets it, and the operator merges there
(the branch has no protection or ruleset). The finished page goes to `master` as one PR for CLB.
Keep the branch current with `master` before that PR. 01 (#135) and 02 (#136) are merged
there; the combined branch was verified green (lint, 601 tests, build) on 2026-09-28.

Source spec: `docs/cqc/spec-cqc-fe-release-1.md`. The API contract is in
`docs/cqc/backend-contract.md` and the decisions in `docs/cqc/decisions-2026-09-26.md`.

Dependency graph (`depends:` is enforced; a blocker must be `done`, that is merged):

```
01 drawer-keeps-keyboard-focus ─────────────┐
02 gated-content-quality-page ── 03 list ───┼── 06 drawer-checks ── 07 drawer-navigation ── 09 technical-tab
                                  │  │  │    │        │  │  │
                                  │  │  └────┼────────┘  │  └── 08 evidence-tab
                                  │  04 row-facts ───────┼── 10 trigger-checks
                                  │          └───────────┴── 11 live-refresh
                                  05 filters ────────────────── 12 resolve-validate (also needs 06)
```

Frontier at start: 01 and 02.

## Before the first dispatch (operator)

1. Land `docs/cqc/` (without `docs/cqc/prototype/`), `.agents/skills/cqc-frontend/`,
   `.agents/design-system/` and the `AGENTS.md` change on `master`. The factory
   cuts every worktree from `origin/master`, so the agent cannot see untracked files.
2. ~~Add the Statsig legacy JS SDK before `cqc-fe-02`.~~ Dropped 2026-09-28: the
   CMC ships internal features without a Statsig gate (spec D-1 amended).
3. `adw-store-01-tickets-dir` must be merged in adw-factory, and `targets/cmc.json`
   must set `"ticketsDir": "~/adw/backlog/cmc"`. Until then the factory reads
   `<repo>/tickets` only.

## Running

```bash
cd ~/personal_project/adw-factory
TARGET=cmc just next                          # what is runnable now
TARGET=cmc just run-codex cqc-fe-01-drawer-keeps-keyboard-focus
```

Every ticket pins `model: gpt-5.6-sol`, run through the Codex CLI's ChatGPT
login (the Go1 subscription in `~/.codex`). Gates (tslint, jest, build; the build type-checks)
run in Docker `node:14.21.3`.
