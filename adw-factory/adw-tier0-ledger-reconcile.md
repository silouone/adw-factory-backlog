---
id: adw-tier0-ledger-reconcile
type: chore
status: done
priority: 3
created: 2026-09-10
depends: []
attempts: []
---
# Tier 0 — reconcile the adw-m5 epic ledger

## Context

First ticket the factory runs **against its own repo**
(`targets/adw-factory.json`), under the `tickets/README.md` amendment of
2026-09-10. Roadmap source: `ai_docs/2026-09-10-deep-state-and-roadmap.md`
Part 3, Tier 0 items 0.2 / 0.3 / 0.4. Every edit below is spelled out here
so this ticket is self-contained — do not go re-derive it from the roadmap.

Purely a markdown ledger edit. **No `src/` or `test/` file is touched.**

## Deliverables

Edits to `tickets/adw-m5.md`, `tickets/adw-m5-03-remote-ci-round.md`, and one
new file `tickets/adw-m5-07-remote-ci-round-live-bar.md`.

## Requirements

**0.3 — the epic's `children:` list is missing an entry, and its table is
missing a different one.**

- [x] In `tickets/adw-m5.md`, add `adw-m5-06-remote-capture-parity` to the
      `children:` frontmatter list, after `adw-m5-05-settlement-bound`.
      Then add `adw-m5-07-remote-ci-round-live-bar` as the last entry.
- [x] In the same file's `## Children` table, add the missing
      `adw-m5-05-settlement-bound` row (it is in `children:` but absent from
      the table — the mirror image of the m5-06 bug) and an
      `adw-m5-07-remote-ci-round-live-bar` row. Keep the table rows in the
      same order as `children:`.

**0.2 — the epic status contradicts its children.**

- [x] `tickets/adw-m5.md` frontmatter reads `status: queued` while four of
      its children are `done`. Set it to `status: in-progress`. It is NOT
      `done`: `adw-m5-06-remote-capture-parity` is still `queued` (it is the
      only red §6 metric, Tier 1.1) and the new m5-07 is queued.

**0.4 — no ticket lingers `in-progress`.**

- [x] `tickets/adw-m5-03-remote-ci-round.md` is offline-complete (634 tests
      green) and owes only its **live** bar — box 128, "Live verify", which
      by construction requires `--isolation remote` against a real E2B
      sandbox and therefore cannot be discharged locally. Set its
      `status: done`, tick box 128's `- [ ]` to `- [x]`, and append a short
      `## Carve-out (2026-09-10)` section recording that the live bar moved
      to `adw-m5-07-remote-ci-round-live-bar` — code-complete is not
      live-verified, and the ticket must say so rather than imply it.
- [x] Create `tickets/adw-m5-07-remote-ci-round-live-bar.md`:
      `type: chore`, `status: queued`, `priority: 3`, `created: 2026-09-10`,
      `epic: adw-m5`, `depends: []`, `attempts: []`. Body carries the owed
      bar verbatim from m5-03's "Still owed" section: a remote-run scratch
      ticket with a deliberately CI-red PR completes one in-sandbox repair
      round on the resumed session. Mark it plainly as **operator-gated and
      billed** — E2B is selected declaratively by a human via
      `--isolation remote`; it is never the default.
- [x] In `tickets/adw-m5.md`, under `## Exit criteria`, add one sentence:
      the epic cannot reach `done` until `adw-m5-07`'s billed live bar runs.
      This is INTENTIONAL, not an oversight — `tickets/README.md` says an epic
      is done when every child is done, and m5 now carries a permanently
      deferred, billed child. A future reader must find that stated, not infer
      it.

## Verify

- `bun run lint && bunx tsc --noEmit && bun test` — green (unchanged; this
  ticket touches no code, so the gates prove only that it broke nothing).
- `tickets/adw-m5.md` `children:` and its `## Children` table list the same
  seven ids in the same order.
- `grep -c '^status: in-progress' tickets/*.md` returns 0.

## Out of scope

- Re-typing the legacy `type: epic` / `type: feature` tickets to the m8 union.
- Any change under `src/`, `test/`, `prompts/`, or `specs/`.
- Tier 1 work (`adw-m5-06`), and actually *running* the m5-07 live bar.

## Result (2026-09-11) — executed BY HAND, not by the factory

Every deliverable is satisfied: m5-03 → `done` with its live bar carved into
`adw-m5-07`, the m5 epic's `children:` and `## Children` table reconciled to
the same seven ids in the same order, epic status `queued` → `in-progress`,
and the exit criteria now state that the epic cannot reach `done` until the
billed bar runs.

**But the factory did not do it, and that was the point of this ticket.** It
was written as the first ticket the factory would run against its own repo.
It never ran, because two defects found later the same day make ANY
self-target run unable to reach a green `gates` node:

- `adw-selfhost-lint-gate` — the lint gate was vacuously red in every
  worktree. FIXED (`84a7371`).
- `adw-gates-config-error-regex` — `bun test` writes test names to stderr,
  two of which contain `command not found`, so `isConfigError` misreads any
  non-zero suite exit as a config error: terminal fail, zero repair rounds.
  OPEN.
- And underneath both: the suite is not green on this host, which is what
  makes the second one fire at all.

So the meta demonstration this ticket existed to produce **has still never
happened**. Marked `done` because the ledger work is genuinely complete;
recorded here that it was hand-executed so the run history is not read as a
successful self-host.
