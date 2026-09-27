---
id: adw-backlog-01-the-backlog-projection
type: feat
status: blocked
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 120, turns: 600}
depends: [adw-cli-01-the-ledger-recipes-ignore-target]
attempts: [{"runId":"adw-backlog-01-the-backlog-projection-1790517794046","branch":"adw/adw-backlog-01-the-backlog-projection","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-backlog-01-the-backlog-projection-1790517794046/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/127","provider":"claude","model":"sonnet","ciRounds":1},{"runId":"adw-backlog-01-the-backlog-projection-1790517794046","branch":"adw/adw-backlog-01-the-backlog-projection","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-backlog-01-the-backlog-projection-1790517794046/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The backlog projection — every ticket of every target, derived, on the wire as JSON

> **Spec authority:** `specs/adw-v1.14-backlog-tab.md` §3 **D2–D5, D7, D9,
> D10**, §4 (the view-model, verbatim) and §5 (routes). `review: false`
> under the hard-gated criterion (`tickets/README.md`, amendment 2026-09-19):
> every Verify bullet is a red test or a gate command. No screen, no CSS, no
> component — that is `adw-backlog-02`.
>
> **Ordering enforced by `depends:`.** `adw-cli-01` ships `ticketsDirOf`;
> this ticket consumes it. Do not re-resolve `<repo>/tickets` here.

## What is being built

`src/web/backlog.ts`: `loadBacklog(targets, views, readTicketDir): BacklogView`
— pure over injected reads, the **only** function in `src/web/` that reads a
ticket file (v1.5 D3, answered yes 2026-09-19, §3 D2). Plus two `GET`
routes in `server.ts` that serialize it.

The queue prototype's `loadQueue` (`src/web/queue.prototype.ts:172`) is the
reference for the join: read `targets/*.json`, resolve each store, parse
with the real `parseTicket`, join `running` against the `RunView[]` the
board already loaded. **Read it, do not import it** — P2 deletes that file.

## Requirements

- [ ] **R1 — the view-model is §4, verbatim.** `BacklogView`,
      `BacklogProject`, `BacklogRow`, `BacklogDep`, `BacklogState` exported
      from `src/web/backlog.ts` with exactly the fields §4 lists. No wall
      clock in any field: `lastAt` is the epoch from the runId
      (`runStartEpoch`), never a rendered age.
- [ ] **R2 — the five states + hard/soft + stale (D4).** `running` from
      `views`; `ready`/`waiting` from `depends:` against the **parsed set of
      the same store**; `in-flight`; `blocked`. A dep is `hard` when its
      status is `blocked`/`rejected` or it is absent from the parsed set.
      `stale` is `in-progress` with no running view — **never** `in-review`.
- [ ] **R3 — closed rows are rows (D5).** `done` and `rejected` are returned
      with `closed: true` and `state` absent; `BacklogProject.closed` counts
      them. Hiding is the screen's job, not the projection's.
- [ ] **R4 — the lenient reader (D9).** For every file `parseTicket`
      refuses, a frontmatter-only read produces an `epic` row (with
      `children` progress computed from the parsed set), a `manual` row, or
      a `malformed` row carrying the strict parser's own `errors` text.
      `parseTicket` and `src/intake/` are **not modified**.
- [ ] **R5 — store-less targets are projects (D10).** A target whose
      `ticketsDirOf` does not exist yields `{ hasStore: false, rows: [] }`.
      It is never omitted.
- [ ] **R6 — ordering (D6).** Projects: open-row count desc, then name.
      Rows: priority → hard before soft (waiting only) → oldest `created`
      → id.
- [ ] **R7 — `GET /backlog.json`** returns one `BacklogView`. Token-gated.
- [ ] **R8 — `GET /backlog-events`** (D7): per tick, `stat` each store dir;
      re-read only when an mtime moved; recompute `running` from the journal
      tail the board's `/events` already maintains; push the serialized
      `BacklogView` **only when it differs** from the last frame sent on this
      connection. Same `TimerSeam` idiom as `/events`.
- [ ] **R9 — read-only by construction.** Both routes are `GET`;
      `server.test.ts`'s no-mutating-literal guard stays green unmodified.

## Verify

- [ ] Red test (Art. I, §8 criterion 1): a fixture store of N files — some
      valid, one epic, one manual, one malformed — yields exactly N rows, and
      a target with no dir yields one `hasStore: false` project. RED today.
- [ ] Red test (§8 criterion 2): two `waiting` rows, one whose dep is
      `blocked`, one whose dep is `in-progress`, carry `hard: true` and
      `hard: false` respectively.
- [ ] Red test: `in-progress` with no running view → `stale: true`;
      `in-review` with no running view → `stale: false`.
- [ ] Red test: an epic with 6 children, 5 `done` → `children: {done: 5,
      total: 6}`.
- [ ] Red test (§8 criterion 4): with a fixed clock, unchanged mtimes and no
      new journal bytes, two consecutive ticks push **nothing**; touching one
      ticket file's mtime pushes exactly one frame.
- [ ] Red test: the serialized frame contains no `%`, no `m `/`s` duration
      string, no `ago`.
- [ ] Red test (§8 criterion 3): for the adw-factory fixture, the `ready`
      set equals the set `just next` lists above its `deps unmet` line.
- [ ] `grep -rn "readFileSync\|readdirSync" src/web/backlog.ts` shows the
      reads only through the injected `readTicketDir` parameter — the
      default binding lives in `server.ts`, like `board.ts`'s `readJournal`.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

- The screen, the nav, the panel, the hero, any CSS (`adw-backlog-02`).
- GitHub reads (an `in-review` ticket whose PR merged is `adw sync`'s job).
- Per-ticket spend.
- `ticketsDir` (v1.4). Deleting the queue prototype (P2 R4).
