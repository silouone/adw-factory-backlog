# Amendment v1.11 — the view layer becomes components, and the wire carries data

> **Status:** proposed 2026-09-18, from the defect investigated the same day
> (`ai_docs/2026-09-18-web-fe-render-thrash.md`) and the stack decision it
> forced (`ai_docs/2026-09-18-web-fe-framework-decision.md`).
> **Depends on:** `specs/adw-v1.5-operator-console.md` §3 **D1**, answered
> 2026-09-18 — without that answer nothing here is buildable.
> **Amends:** `adw-v1.2-live-view.md` decision 11 (already reversed by v1.5 D1)
> and the `/events` + `/run-events` payload contract established by
> `adw-fe-08-live-sse` and `adw-fe-16`.
> **Binding:** `constitution.md` — unchanged. Article I and Article IX apply
> unmodified; §7 below says what they mean for a component.
> **Does NOT amend:** `adw-v1.7-run-panel.md` or `adw-v1.10-linear-timeline.md`.
> Every view *contract* those specify — the panel, the rail, the
> one-entry-per-box rule, the metric tri-state, the linear axis — is
> **preserved verbatim**. This amendment changes how a view is expressed, never
> what it says.

## 1. Trigger

Two defects, both measured 2026-09-18 against a live `adw web`:

- **The fragment is the page.** Both screens assign a server-rendered HTML
  string into one container's `innerHTML`, destroying 5,158 DOM nodes at a
  time. Every piece of operator state dies with them: `<details>` expansion,
  window scroll, inner scroll, text selection, keyboard focus, `:hover`.
- **The change-detector is defeated by a clock.** `if (body !== lastSent)` is a
  string compare over markup containing `19m 42s` and `left:29.284%`. Measured:
  **11 pushes in 10 s against 0 new journal bytes**, at 302.6 KB per board
  frame.

The first defect has been patched four times — `adw-usage-01` (PR #61) hoisted
the project dropdown's `open` flag into a closure, `adw-usage-02` did it again
for the usage chip, `render-run.ts` does it a third time for the dock scope,
and **`adw-fe-23` R4 is queued to do it a fourth time** for zoom and
`scrollLeft`. That strategy is O(one hand-written patch per interactive
widget), and it structurally cannot reach scroll position, text selection or
focus, because those are properties of nodes that no longer exist.

**This amendment closes the class, rather than patching the fifth instance.**

## 2. What changes

### 2.1 The wire carries view-models, not markup

Today `/events` and `/run-events` push rendered HTML. They will push
**serialized view-models** — the `RunView`, `CardSummary`, `GanttView` and
`UsageRead` interfaces that `src/web/` already exports and already tests.

This is the decision that protects the migration's economics (§3), and it is
**binding**: the wire carries neither HTML nor raw journal records. Derivation
stays server-side.

### 2.2 The wall clock stops crossing the wire

`elapsed` becomes a **client-side ticker over `startedAt`**. The server pushes
only when journal bytes actually arrive.

This retires defect two structurally rather than by rounding: there is no
seconds-resolution string in the payload to defeat the diff, and no float axis
geometry either — the payload carries **timestamps**, and the live geometry is
derived from them on the client.

> **Exactly one clock reader exists on the client**: a single `now` signal
> owned at the application root, ticking at 1 Hz. Live geometry and `elapsed`
> are `computed()` over it, and reach components **as props**. No component
> calls `Date.now()`, and §7's Article IX rule is not softened by this
> paragraph — it is what makes it implementable.

> **Consequence worth stating plainly:** an idle live run will push **nothing**.
> Today it pushes 302 KB/s.

### 2.3 Views become Preact components

`render.ts` and `render-run.ts` are replaced by TSX components under
`src/web/ui/`. Per v1.5 D1: Preact + `@preact/signals`, bundled by `bun build`,
no second toolchain.

## 3. What survives — and why that is the point

Measured across `src/web/` (7,384 lines) and `test/web/` (12,536 lines):

| | lines | fate |
|---|---|---|
| `projection` `board` `card` `gantt` `timeline` `drawer` `usage` `metrics` `capture` `chain` `run-view` `runs` `tailer` `pricing` `run-summary` | **3,446** | **untouched** — they emit zero HTML |
| their tests | **7,602** | **untouched** — they assert on derived data |
| `render.ts` + `render-run.ts` | 2,448 | replaced by components |
| `render.test.ts` + `render-run.test.ts` + `usage-parity` + parts of `server.test.ts` | ~4,934 | rewritten as component tests |
| `server.ts` | 639 | modified — serialize, don't render |
| `queue.prototype.ts` | 851 | deleted (already marked THROWAWAY) |

**~47 % of the source and ~61 % of the tests carry over verbatim.** The
derivation layer is already pure and already isolated — only two imports cross
into `src/web/` from the rest of the factory (`cli.ts` → `startWebServer`,
`kill.ts` → `EXCLUDED_DIRS`).

## 4. Phases

Each phase ends green on `bun run lint && bunx tsc --noEmit && bun test`. No
phase is allowed to leave both a rendered and a component path live for the
same screen beyond its own completion.

**P0 — foundation (hand-built).** `bun build` wiring, `tsconfig` JSX config,
the `@testing-library/preact` + `happy-dom` test seam, the serialization
boundary, and **one** converted component end-to-end as the reference the
later tickets copy. Hand-built deliberately: this is where the conventions get
decided, and every later ticket inherits them.

**P1 — the wire.** `/events` and `/run-events` emit view-model JSON; the client
subscribes and holds it in signals. The board still renders server HTML at
this point; this phase is the contract change alone, provable by a test that
the payload parses to a `RunView[]`.

**P2 — the board.** `render.ts` → components. Retires `renderGridBody`,
`filterScript`, `renderGrid`, and the `?queue=` prototype branch.

**P3 — the run screen.** `render-run.ts` → components. Retires the dock IIFE,
`showScope`, the md/raw `dataset.done` memo, and `DOCK_SPLIT_MARKER`.

**P4 — parity sweep.** Delete the retired modules and their tests. Confirm the
v1.7 and v1.10 view contracts still hold.

## 5. Sequencing against v1.10 — read before dispatching either ticket

`adw-v1.10-linear-timeline.md`'s two tickets land on opposite sides of this
work, and getting it wrong wastes a full ticket:

- **`adw-fe-22-the-gantt-axis-is-not-time` — before P2. Unaffected in
  substance.** Its value is in `timeline.ts`'s `layoutTimeline` (the
  `--- warp ---` block), a **surviving** derivation module with surviving
  tests. Only its `laneShare` removal touches `render-run.ts`. Land it before
  P2 so the migration inherits a correct axis instead of porting a warped one.

  > **Status note, 2026-09-18:** its first run
  > (`…-1789719265397`) ended **blocked** at `review-fix` — *"the fix broke
  > gates — gate `test` regressed, exit 1"* — and the ticket is now
  > `status: blocked`, not `queued`. It needs a retry, not a re-plan.
  >
  > **Read the reviewer's top finding as evidence for this amendment.** It is
  > that `blocksOf` in `test/web/render-run.test.ts` parses boxes with a regex
  > over `class="nd …" style="left:…%;width:…%"`, cannot match the new marker
  > markup, and therefore makes an `adw-fe-19` regression test **pass
  > vacuously**. That helper — and the file it lives in — are deleted by P4.
  > A repair round spent on markup-regex test infrastructure is exactly the
  > cost this migration removes.

- **`adw-fe-23-zoom-and-scroll-the-run-timeline` — after P3. Re-sequence it.**
  Nearly all of its blast radius is in the layer being deleted: it restructures
  `renderLanes`'s markup in `render-run.ts`, rewrites that module's client
  IIFE, and updates `render-run.test.ts`'s regex markup parsers. Worse, its
  **R4 is the band-aid a fifth time** — *"capture `scrollLeft` and the zoom
  class before the `innerHTML` assignment and restore after, the same shape as
  the existing `showScope(active)` call."*
  **Under P3 that requirement becomes free**: zoom and scroll are component
  state and simply survive. Landing fe-23 first means paying for R4, then
  deleting it.

> fe-23 was set to `status: blocked` on 2026-09-18 for this reason; widen its
> `depends:` to the P3 ticket once that ticket exists. Left `queued` it would
> have been picked by `just next` today.

### `adw-fe-24` — dispatch it now. It is a down-payment on P1, not a casualty.

`adw-fe-24-the-live-tick-erases-every-prompt-from-the-panel` (`priority: 1`,
filed 2026-09-18) is the **sixth** instance of this amendment's defect class,
and the most damaging: the live fragment is *lossier than the page it
replaces*. `handleRunEvents` passes `new Map()` for `drawers`
(`server.ts:595`), so a tick renders **0** prompts where the static page
rendered 2 — and the wholesale swap then erases them about one second after
load. Measured in that ticket: 70,740 bytes and 2 prompts static, versus
15,146 bytes and 0 prompts on the live path.

**It is not blocked, and it must not wait for P3.** Unlike fe-23, its fix
lands in `server.ts` and `run-view.ts` — modules this amendment **keeps** —
and its "Out of scope" already excludes the client-side DOM band-aid. Better
still, making the live path carry prompt content **is** what §2.1 requires of
the wire: a view-model that omits what the static render included is not a
view-model. **Treat fe-24 as P1's first slice**, and let P1 generalise it.

> That this class produced a `priority: 1` defect while the amendment closing
> it was being written is the argument for the amendment.

**One caveat on dispatching fe-24 ahead of P1.** Its file-read cost is already
bounded — the ticket mandates a per-stream memo keyed on the sidecar path,
read-and-hashed *at most once per SSE connection*, justified from
`makePromptSink`'s write-once sequence. What it does **not** bound is the
**payload**: restoring prompts to the live fragment takes a run-screen push
from ~15 KB to ~70 KB, and the 1 Hz driver is still in place until §2.2 lands
(the operator chose 2026-09-18 to skip the interim clock-quantisation fix).
So fe-24 is correct and should ship, but it makes the thrash *heavier* until
P1 retires the driver underneath it. **P1 should follow it closely**, and
neither should be left half-done.

## 6. Decisions

**D1 — the wire format is view-model JSON.** Settled by §2.1. Raw journal
records were rejected: they move derivation (and 7,602 lines of tests)
client-side for no gain, since the server has already read the journal.

**D2 — one bundle, built at launch, served from memory.** `adw web` is
loopback-bound and per-launch-token'd, so the bundle is produced on `adw web`
start and served from the existing server. No build artifact is committed, no
watch mode, no dev server.

> **Verified 2026-09-18, not recalled.** `Bun.build({entrypoints, target:
> "browser", minify: true})` called *programmatically* returns an in-memory
> output whose `.text()` is directly servable: **11 KB in 3 ms** for a Preact
> entry. The route set grows by one **`GET`** (`/bundle.js`), token-gated like
> every other route, which keeps "read-only by construction" intact —
> `server.test.ts`'s structural guard asserts the source contains no
> `"POST"`/`"PUT"`/`"DELETE"`/`"PATCH"` literal, and a `GET` branch does not
> trip it.

**D3 — the CSS ports verbatim, first.** The 472 lines in the two `css()`
functions are plain CSS and are the design v1.7/v1.10 specify. They move
unchanged in P0 and are **not** redesigned here. A restyle is v1.5's business,
not this amendment's.

**D4 — component tests replace markup-string assertions.** ~4,934 lines are
rewritten, not ported. They assert rendered output via
`@testing-library/preact`, not regex over an HTML string.

## 7. What Article I and Article IX mean for a component

- **Article I** — the red test comes first, and it is a *component* test.
  Verified 2026-09-18 that this is satisfiable on the existing runner: a
  broken component turns `bun test` red with a real diff
  (`Expected "3m 25s" / Received "BROKEN3m 25s"`) in 240 ms, with no second
  test runner.
- **Article IX** — a component is a **pure function of its props**. All
  derivation stays in the existing pure modules; a component may not read the
  clock, the filesystem or the network. `elapsed` (§2.2) is the one live value,
  and it enters as a prop from a single signal owned at the root. Verified that
  `exactOptionalPropertyTypes: true` enforces through JSX props (`TS2375`), so
  the repo's strictest flag gains reach rather than losing it.

## 8. Non-goals

Unchanged from v1.2/v1.5: no router agent, no watcher daemon, no scheduled
runs, no queue draining, **no mutating route**. Additionally, and specific to
this amendment: **no redesign**. Every screen after P4 looks like it did
before. This is an architecture change whose success criterion is that the
operator notices nothing except that the page stopped fighting them.

Also out of scope: SSR, routing libraries, a state-management library beyond
`@preact/signals`, and any CSS framework.

## 9. Success criteria

1. `bun run lint && bunx tsc --noEmit && bun test` green at every phase
   boundary, with no second toolchain added.
2. **An idle live run pushes zero bytes** over a 10 s window — the direct
   inverse of the 11-pushes/0-bytes measurement that triggered this.
3. A user-collapsed section stays collapsed, and inner scroll, text selection
   and keyboard focus survive, across ≥10 live ticks. (Verified in the spike:
   0 childList mutations on the section across 9 clock ticks; 2 mutations from
   the 2 real events.)
4. The v1.7 and v1.10 view contracts still hold after P4.
5. `queue.prototype.ts` and the `?queue=` branch are gone.

## 10. Requires an operator decision

None outstanding. v1.5 D1 is answered and this amendment is its consequence.
**v1.5 D3** (may `src/web/` read the ticket store?) is still open but does not
block anything here — it gates `adw-console-01`, not the render architecture.
