# Amendment v1.14 — the backlog tab: every ticket, every project, every state, on one screen

> **Status:** proposed and **decided 2026-09-19** in one operator grilling
> session (two rounds, twenty decisions, all answered). Ready to build.
> **Answers:** `adw-v1.5-operator-console.md` §3 **D3** — *may `src/web/`
> read the ticket store?* — **yes**, under §3 D2 below.
> **Supersedes:** `tickets/adw-console-01-operator-console.md`. Its "filter
> runs by target" shipped in `adw-fe-14`; its "dependency visibility across
> tickets" *is* this amendment. The ticket is closed `rejected: superseded`.
> **Depends on:** `adw-render-01-foundation` (**done**, P0 of
> `adw-v1.11-render-architecture.md`) — the component conventions this screen
> is born into. Does **not** depend on P1–P4.
> **Defers:** `adw-v1.4-ticket-store.md` — untouched, still proposed. This
> amendment reads `<repo>/tickets` through one seam v1.4 can later widen.
> **Binding:** `constitution.md` — unchanged. Art. IX applies to the
> projection (pure, zero I/O) and to the component (a pure function of its
> props, `src/web/ui/README.md`).
> **Implemented by:** `adw-cli-01-the-ledger-recipes-ignore-target` (chore),
> `adw-backlog-01-the-backlog-projection` (feat),
> `adw-backlog-02-the-backlog-screen` (feat),
> `adw-backlog-03-the-backlog-screen-is-legible-and-usable-4b17e2` (feat,
> the design pass; see §10 2026-09-27).

## 1. Trigger

The operator, 2026-09-19:

> *"I can't go through all tickets in markdown in the repos. Experience proved
> that I forget some and leave them behind. I need a dedicated interface for
> that: the BACKLOG, with filter, with per-project view. I need to see waiting
> tickets, queue, blocked — ALL states really. We thought of a modal but this
> deserves a real tab."*

Measured the same day, the forgetting is structural, not a lapse:

1. **The web view is a view of runs, not tickets.** Every pixel of `adw web`
   derives from `runs/*/journal.jsonl`. A ticket that has never run has no
   journal, so it does not exist on the board. `clens-010`, `-012` and `-013`
   are `queued` and invisible.
2. **The CLI has no per-project view either.** `just next`, `just tickets`,
   `just ticket` and `just in-progress` hardcode `tickets/` and ignore
   `TARGET`. `TARGET=clens just next` prints adw-factory's backlog.
3. **`just next` conflates two kinds of waiting.** `⇠ waits on X` is printed
   identically whether X is `in-progress` (clears itself) or `blocked` (never
   will). The second kind is the one only a human can move, and it is not
   distinguishable from the first.
4. **The operator-owned tickets are unlisted by design.** 14 files fail
   `parseTicket` — 13 `type: epic`, 1 `type: manual` — correctly, because they
   have no lane. An epic whose children are all done but still reads `queued`
   (`adw-m5`, named in the README) is a forgotten ticket by construction.
5. **The 2026-09-15 queue drawer prototype** (`src/web/queue.prototype.ts`,
   `QUEUE-PROTOTYPE-NOTES.md`) answered the wrong container question and the
   right data questions. Its verdict section was never filled in. Its
   findings — the five-state model, hard-vs-soft blockers, the target→store
   join, and "a drawer frozen at load is worse than none" — are adopted here;
   the prototype itself is deleted by `adw-render-03` P2 R4, unchanged.

## 2. What this amendment adds

A third screen, **`GET /backlog`**, beside the board (`/`) and the run screen
(`/run`). It shows **every ticket of every configured target**, grouped by
derived state, filterable by project, with a read-only detail panel per
ticket. Its guarantee is the inverse of the board's: **nothing that exists in
a ticket store is absent from this screen** — not a queued ticket with no run,
not an epic, not a target with no store, not a file that fails to parse.

## 3. Decisions

**D1 — the job.** The tab is an *inventory* first and an *exception list*
second. Every non-closed ticket of every project is always on screen; the
rows only a human can move — `blocked`, and `waiting` on a hard blocker —
sort to the top. "What can I dispatch next" is answered by a hero slot, not
by hiding everything else.

**D2 — the web layer reads the ticket store (v1.5 D3 = yes).** Through
exactly one function: `loadBacklog(targets, views, readTicketDir)` in
`src/web/backlog.ts`, pure over injected reads, tested like `board.ts`. No
other module in `src/web/` may read a ticket file. The coupling is one
function, not a habit.

**D3 — the store is `<repo>/tickets`, resolved through one seam.**
`ticketsDirOf(target)` in `src/targets/loader.ts` returns `<repo>/tickets`
today. `adw-v1.4` lands into that function later, if ever; nothing here waits
on it.

**D4 — the five-state model, plus `stale`.** Derived per row, never read
from `status:` alone:

| state | rule |
|---|---|
| `running` | a `RunView` with `state === "running"` names this ticket |
| `ready` | `queued` and every `depends:` entry is `done` — what `just next` offers |
| `waiting` | `queued` and at least one dep is not `done` |
| `in-flight` | `in-progress` or `in-review` |
| `blocked` | `status: blocked` |

Each unmet dep is **hard** (dep is `blocked`, `rejected`, or absent from the
parsed set) or **soft** (dep is `queued`/`in-progress`/`in-review`). Hard is
a wall; soft clears itself. This distinction is the single thing the tab
knows that the CLI does not.

`stale` is a flag, not a state: **`in-progress` with no live run behind it**
— the factory's own forgotten ticket (a claim of a run that died). It does
**not** apply to `in-review`: a PR awaiting a human has no run and is not
stale. (An `in-review` ticket whose PR is already merged is `adw sync`'s
job, not this screen's — it needs a GitHub read this amendment does not add.)

**D5 — closed tickets are hidden by default, never uncounted.** `done` and
`rejected` sit behind one "closed" toggle, off by default, with distinct
glyphs. Every project header always shows `N open · M closed` so the size of
the closed pile is visible without opening it.

**D6 — state-major sections, project as a filter.** The prototype measured
state-major (variant B) as the most legible at today's volume, and
project-major buries the ready rows under the running ones. The project
filter reuses the board's dropdown vocabulary — same gesture, same labels —
and a one-line count strip per project (`clens · 1 running · 3 ready · …`)
answers "what waits where" without scrolling. Inside a section: priority →
hard before soft (`waiting` only) → oldest `created` → id. Age is the
forgetting signal, so oldest first.

**D7 — live, with a throttled store read and an honest push gate.** The
screen has its own `GET /backlog-events`. Per tick the server `stat`s each
ticket dir and re-reads only when an mtime moved; the journal-derived
`running` set rides the tail the board already pays for. The serialized
view-model is diffed and pushed **only on change** — an idle backlog pushes
zero bytes. This is the v1.11 §2.1/§2.2 contract from day one: the wire
carries view-model JSON, no HTML, no wall clock (timestamps only).

**D8 — born as components.** The screen is a Preact component under
`src/web/ui/`, following P0's fixed conventions. It ports no legacy code,
so it does not wait for P1–P3 — it is the second proof of P0. Nav is a
`Board · Backlog` strip in the component plus one `<a>` in the legacy board
header, which P2 deletes when it converts the board.

**D9 — operator-owned tickets are rows, not omissions.** `parseTicket` stays
strict (it guards dispatch). The tab adds a deliberately **lenient**
frontmatter-only reader, used *only* for files the strict parser refuses:
an `epic` row shows its `children` progress (`adw-m5 · 5/6 done`); a
`manual` row is labelled operator-owned; anything else is a `malformed` row
carrying the strict parser's own error text. The intake contract is not
touched.

**D10 — a target with no ticket store is listed, not skipped.** 9 of 11
targets have no `tickets/` dir today. Each appears as a project row reading
"no ticket store" — an absence you can see is an absence you will not forget.

**D11 — the row and the panel.** A row: state glyph · short id (prefix
stripped, full id on hover) · title (the `# ` heading) · lane · priority ·
dep chips coloured hard/soft · `attempts · last outcome · age` · PR link if
the last attempt has one · `review: off` marker · copy button. Clicking a
row opens a **side panel on the tab** (component state, so it survives every
tick): every frontmatter fact, the rendered markdown body, the `attempts:`
ledger with each run linked to `/run?id=`. The hero is the top `ready` row of
the active filter; its copy button copies the **full dispatch line**
(`TARGET=clens just run clens-010-…`) — the target is exactly the part the
CLI gets wrong today. Per-ticket spend stays in the panel (it is a journal
read per attempt); the row carries counts only.

**D12 — the CLI is fixed first, separately.** The TARGET-blind recipes are a
ten-line chore, they would have surfaced the clens tickets last week, and
landing them first gives the tab a CLI to be checked against.

## 4. The view-model

Exported from `src/web/backlog.ts`, serialized verbatim on the wire:

```ts
type BacklogState = "running" | "ready" | "waiting" | "in-flight" | "blocked";

interface BacklogDep { id; status; met: boolean; hard: boolean }

interface BacklogRow {
  id; project; title;
  kind: "ticket" | "epic" | "manual" | "malformed";
  lane?: "chore" | "bug" | "feat"; priority?: 1 | 2 | 3;
  status: string;            // the frontmatter value, verbatim
  state?: BacklogState;      // absent for closed rows and non-ticket kinds
  closed: boolean;           // done | rejected
  stale: boolean;
  created?: string;
  deps: readonly BacklogDep[];
  attempts: number; lastOutcome?: string; lastRunId?: string; lastPr?: string;
  lastAt?: number;           // epoch ms from the runId — a timestamp, never "2h ago"
  review?: boolean; model?: string;
  children?: { done: number; total: number };   // epic rows
  parseError?: string;                          // malformed rows
}

interface BacklogProject {
  name; hasStore: boolean; storeDir?: string;
  rows: readonly BacklogRow[];
  counts: Record<BacklogState, number>; closed: number;
}

interface BacklogView { projects: readonly BacklogProject[] }
```

Projects sort by open-row count descending, then name — the board's own rule.

## 5. Routes

| route | returns | owner |
|---|---|---|
| `GET /backlog.json` | one `BacklogView` snapshot | backlog-01 |
| `GET /backlog-events` | SSE, `BacklogView` JSON, pushed on change | backlog-01 |
| `GET /backlog` | the screen shell + `/bundle.js` | backlog-02 |

All token-gated like every other route. All `GET` — `server.test.ts`'s
no-mutating-route guard is untouched. **The console stays read-only.**

## 6. Sequencing

```
adw-cli-01 (chore)  ──►  adw-backlog-01 (feat, pure)  ──►  adw-backlog-02 (feat, screen)
                                                              ▲
                              adw-render-01 (P0, done) ───────┘
```

Independent of `adw-render-02..05`. When P2 converts the board it inherits
the nav strip and deletes the legacy anchor; when P4 deletes the queue
prototype nothing here notices — this amendment never imports it.

## 7. Non-goals

Unchanged: no mutating route, no dispatch from the browser, no router agent,
no queue draining. Additionally: no GitHub read (PR merge state is
`adw sync`'s), no per-row cost (panel only), no change to `parseTicket` or
the intake contract, no `ticketsDir` (v1.4 stays proposed), no redesign of
the board or run screen.

## 8. Success criteria

1. Every file under every configured target's ticket dir is represented by
   exactly one row, or the target by one "no ticket store" line — proven by a
   test that counts files and rows.
2. `waiting` on a `blocked` dep and `waiting` on an `in-progress` dep render
   distinguishably. RED today, everywhere.
3. `just next` and the tab's `ready` set agree for every target, after
   `adw-cli-01`.
4. An idle backlog pushes 0 bytes over 10 s of `/backlog-events`.
5. The detail panel stays open, at the same scroll, across ≥10 live ticks.
6. `bun run lint && bunx tsc --noEmit && bun test` green, no second toolchain.

## 9. Requires an operator decision

None outstanding. All twenty decisions were put to the operator on 2026-09-19
and answered; this document is their record.

## 10. Change log

- [DECIDED] 2026-09-19 — the backlog tab; v1.5 D3 answered yes;
  `adw-console-01` superseded.
- [AMENDED] 2026-09-27 — the design pass, `adw-backlog-03` (the operator
  picked prototype variant A, "ledger", from `.proto/backlog/`). §7's "no
  redesign" meant the board and run screen. This redesign covers the backlog
  screen only, in the board's own language (`adw-fe-15`). Changes:
  - **§4 view-model.**
    - `BacklogRow.reason?`: the verbatim `run-end.reason` of the row's last
      run. It is joined in `loadBacklog` from the `RunView` whose `runId` is
      `lastRunId`, and is absent when that run was pruned or never ended.
      It is never fabricated.
    - `BacklogRow.drift?`: a named mismatch between the ledger and reality
      (`in-progress, but no run was ever dispatched`, `in-progress, no live
      run`, `in-review, no PR recorded`). It is time-free, so §8 criterion 4
      (an idle backlog pushes 0 bytes) still holds. `stale` stays as it was.
    - `BacklogProject.provider`: the target's resolved provider (`claude`
      when absent). It feeds the per-row dispatch line (`just run-codex`).
  - **D1.** "Every non-closed ticket … always on screen" now holds *by
    default*. Filters (project, state, lane, priority, free text) are opt-in,
    and they and the selection live in the URL query (`proj st lane p q sel
    closed`), merged next to `token`.
  - **§8 criterion 1.** It still holds at the projection level: one
    `hasStore:false` project per target. On screen, those targets collapse
    behind one "+n targets without a store" toggle, which reveals the same
    per-target line.
  - **§8 criterion 2.** On the row, it is carried by the why-line's tone
    ("waits on X (blocked)" is bad, "waits on X (in-progress)" is warn). The
    dep chips move into the panel.
  - **Shared markdown.** `mdToHtml` joins indented continuations of list
    items and merges consecutive quote lines (table rows excepted). It also
    renders `- [ ]` / `- [x]` as task items (`rp-task`, `rp-task-done`,
    `rp-cb`), which the run screen's CSS styles too.
- [AMENDED] 2026-09-27 — **one unified header** (`adw-backlog-03` R11,
  requested by the operator after reviewing PR #135: "the board/backlog
  changed place between the tab"). It replaces the separate board and
  backlog navs D8 introduced, and fulfils §6's "the board inherits the nav
  strip and deletes the legacy anchor".
  - **The header.** `<AppHeader/>` shows the brand, then the Board and
    Backlog tabs, then the active screen's context. The tabs are in the same
    place on every screen, including the run screen, where they are plain
    links and neither is active.
  - **Switching.** Board ⇄ Backlog switches client-side: the header element
    persists, and only its context and the screen below it change. Routes
    stay path-based (`/`, `/backlog`), and `?id=` still selects the run
    screen, so the render-04 plan's D1 (the page is chosen by the URL, `?id=` → run)
    and v1.11 D2 (one bundle) hold.
  - **Streams and URL.** Exactly one SSE stream is live: the active route's.
    Each route keeps its own query string, and Back/Forward switch tabs.
  - **One stylesheet.** `/` and `/backlog` serve the same `appCss()`. The
    backlog's rules are scoped under `.bl` wherever they touch a board class.
- [AMENDED] 2026-09-28 — **the dependency graph** (operator, three prototype
  rounds on real data in `.proto/backlog-graph/`, never committed). The
  operator's reference was their own terminal wave plan, e.g. `WAVE B2 …
  04 GET /content (needs 02+03)`, plus its critical-path line. The operator
  signed it off as "GOOD ENOUGH for a v1".
  Implemented by `adw-graph-01` … `adw-graph-06` (ids with uid suffix in the
  store).
  - **G1 — waves are over OPEN tickets only.**
    - A closed ticket is wave 0.
    - An open ticket's wave is `1 + max(wave of its OPEN deps)`, so wave 1
      means "fires as soon as nothing open is ahead of it".
    - A dep that is not a row of the project is an *outside* node, wave 0,
      open iff `!met`.
    - A cycle does not hang: the guard assigns wave 1.
  - **G2 — done history is pruned.** A closed ticket is a node only if some
    open ticket of the project depends on it directly.
  - **G3 — only edges INTO an open ticket are drawn.**
    - A done ticket's own deps say nothing about what fires next.
    - Drawn, they cross cards: done→done stays inside one column, and
      open→done points backwards.
    - A closed ticket with an open dep (ledger drift) is badged
      `⚠ n dep open` instead.
  - **G4 — the critical path** is the longest open chain, recorded **edge by
    edge**. Ties go to priority, then id. Marking an edge because both of its
    ends are on the path is wrong: it over-marks.
  - **G5 — live state.**
    - **running:** `state` is `running` or `in-flight`.
    - **blocked:** `state` is `blocked`.
    - A running / in-flight node also carries the row's `lastAt` (elapsed
      time on the card); a blocked node carries the row's `reason` (the
      card's tooltip). Each key is omitted when the row has none.
    - **held:** every open descendant of a blocked ticket (other than a
      blocked one) is *held*, and names its nearest blocking ancestor.
  - **G6 — derivation stays in the projection (R7).** G1–G5 are computed
    server-side and ride on the wire (`BacklogProject.graph`). Only drawing
    geometry, the layered layout, is a pure UI helper.
  - **G7 — the layout is layered, and waves are columns** (prototype variant
    A; B, the wave list, and C, the ego graph, were rejected).
    - Nodes are ordered within a column by barycenter sweeps.
    - A long edge takes a reserved slot in every column it crosses.
    - Edges leave and arrive spread along a card's side.
    - No edge sample may fall inside a card, and a test proves it.
    - Cards are opaque. Dimming fades a card's content, never its
      background.
  - **G8 — the graph is a real drawer, not a floating card.**
    - It docks at the **bottom** by default, or on the **right** by toggle.
    - It is full-bleed to the edge, with no radius, shadow or gap.
    - The ticket panel (`.bl-panel`) docks the same way, flush right and
      full height.
    - The table is **pushed** by the docked sizes, never covered.
    - Its state lives in the URL, merged next to the backlog's own keys:
      `graph` (project), `focus`, `dock`, and `gw` / `gh` (size, only the
      active dock's). A reload restores the drawer.
  - **G9 — the table speaks waves (prototype table variant 1).**
    - A `waiting` row's why-cell reads `W3 need 02 & 03`, or
      `need 04, 05 & 06` for three or more.
    - It lists unmet deps only, each coloured by that dep's state.
    - Each dep is a chip that opens the graph focused on that dep. It never
      selects the row.
    - `02` is the id's sequence part, stripped of the project's common id
      prefix.
  - **G10 — a manual ticket keeps its `depends:`.** The lenient reader
    resolves it exactly like a parsed ticket's. Before this, it returned
    `deps: []` and `cqc-be-11`'s 7 deps vanished.
