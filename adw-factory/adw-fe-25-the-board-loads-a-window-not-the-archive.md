---
id: adw-fe-25-the-board-loads-a-window-not-the-archive
type: feat
status: done
priority: 1
created: 2026-09-21
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-fe-25-the-board-loads-a-window-not-the-archive-1789982332474","branch":"adw/adw-fe-25-the-board-loads-a-window-not-the-archive","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-25-the-board-loads-a-window-not-the-archive-1789982332474/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/105","provider":"claude","model":"sonnet"}]
---
# The board loads a time window, not the whole archive — `1d 3d 7d all` in the header

> Operator, 2026-09-21, on `adw web`:
> *"it is very LONG TO LOAD. can we perform optimisation its becoming an
> issue. one big move we could make is lazy load the run on demand, because
> most of the time I don't need to go back in old run, the last day of run
> would be enough."*
>
> After a three-variant prototype (load-more / range / pages), same day:
> *"B range is the way, but let's remove the slider & let's put that on the
> left of the project choice, in the header."* — then, on the revised
> prototype: *"yes this version is good."*

The board (`GET /` + `/events`) reads **every** banked run on every new
connection and re-sends the **whole** board to every open tab each time any
live run writes a line. With 97 runs on disk that is a ~0.5 s first frame,
846 KB on the wire, and a browser that re-parses and re-renders 95 cards per
second while a run is live. The operator looks at the last day, almost
always. Everything older is loaded, serialised and rendered for nothing.

## Evidence (measured 2026-09-21, 95–97 banked runs, 3 tabs open, 1 live run)

| step | today | why |
|---|---|---|
| `GET /` shell, usage cache cold | **~3 s** (1 ms warm) | `handleGrid` **awaits** `claude -p /usage` (`server.ts:319`) before returning HTML; the client already polls `/usage.json` on its own |
| `/events` first frame, all runs | **0.4–0.6 s** server · **846 KB** | `pollRunDirectories` tails every journal; `buildCardFromRecords` reads every capture; all 95 views + cards stringified |
| same frame, last 24 h only | **~0.1 s** · **147 KB** | 13 of 95 runs. A run outside the window costs **zero** I/O — its epoch is the runId's trailing `-<epochMs>`, so no file is opened |
| each tick a live run appends a line | **846 KB re-sent per tab, per second** | the push gate (`body !== lastSent`) is per **frame**: one run's new line re-serialises the whole board; the client re-parses and re-renders 95 cards |

Journals total 4.8 MB across all runs; the 7 GB under `runs/` is workspaces,
never read here. The cost is **how many runs a connection touches**, not
bytes per run. Curl first-byte on the real server, all runs: 0.40 s / 0.42 s;
prototype, last 24 h: 0.09 s / 0.11 s.

Profile of one full first tick (`tail`+`parseRecord` / `projectRun` /
`buildCardFromRecords`), 95 runs: 49 ms / 2 ms / 509 ms. The capture reads
dominate, and they are paid **per connection**.

## The prototype — the picked version

Throwaway, on the **real** `<Board/>` component and real `runs/`:

```
.proto/lazy/serve.ts     windowed /events + /index.json, zero I/O outside the window
.proto/lazy/entry.tsx    the real <Board/> + the chip group mounted in the header
.proto/lazy/NOTES.md     the question, the measurements, the three variants, the answer
```

```bash
bun .proto/lazy/serve.ts          # → http://127.0.0.1:4322/?days=1
```

**This is the design to ship** (variant B, slider removed, chips in the
header — as approved). The header's right-hand row reads, left to right:

```
[1D] [3D] [7D] [ALL]  ▂▅▃▃▂  since 20 Sept · 13 of 95  │ All projects ▾ │ GREEN 7 │ BLOCKED 5 │ NO RUN-END 1
```

- **Four chips**, styled with the header's own `.chip` rules (uppercase `<i>`
  label, `on` state for the active window). Exactly one is `on`.
- **A per-day histogram**: one bar per day back to the oldest run, newest on
  the left, bars inside the window lit (`--ok` green), the rest dim. Height
  proportional to that day's run count; hover title `"<n>d ago: <k> runs"`.
- **A label**: `since <Mon D> · <in window> of <total>` — or `all <total> runs`.
- Window state lives in the page URL as `?days=1|3|7|all`, reload-stable,
  shareable, carried the same way `?token=` already is. **Default: 1 day.**
- Changing the window **replaces** the board (close the stream, open a new
  one with the new `since`); collapsed sections keep their state through
  the swap because `Section` keys on `group.target` (unchanged).
- "A day" is **24 h from when the page connected**, not a calendar day. The
  window is anchored at connect time so an `EventSource` auto-reconnect
  re-opens the same URL and lands on the same window.

Variants A ("load more", append a day per click) and C (20 runs per page)
were built, tried, and rejected: A is an append the operator did not want,
C splits a ticket's attempts across pages and fights "a card is a ticket".
Their code is deleted from the prototype; `NOTES.md` keeps the comparison.

## Design

**Server — `/events?since=<epochMs>`** (`src/web/server.ts`, `handleEvents`
:502 / `pollRunDirectories` :636). Filter the run-directory listing by
`runStartEpoch(name) >= since` **before** any `tail`, any `parseRecord`,
any capture read, any card build. A run whose id carries no parseable
epoch is included only when no `since` is given (`all`). No `since` =
today's behaviour, byte-for-byte. The frame gains one derivation-free
field, computed from the listing the tick already does:

```ts
index: { readonly total: number; readonly epochs: readonly number[] }
// every banked run's start epoch (parseable ids only), newest first
```

That is what the histogram and the `n of N` label read. It is a few hundred
bytes; it is **not** a second route (the prototype's `/index.json` was a
shortcut).

**Client** (`src/web/ui/`):

- `entry.tsx` reads `?days=` next to `?token=`/`?id=` (the one edge call
  site, Art. IX / plan §1.4), computes `since`, and opens
  `startBoardEvents("/events?since=…&token=…")`. It re-opens on change; the
  `days` signal is the only writer of the URL param.
- `Header.tsx` gains a **slot prop** for the chip group, mounted as the
  first child of `.gh-r` (left of the `details.menu` project dropdown) —
  the same pattern as the existing `usageChip` slot (`Header.tsx:41`), for
  the same reason: `Header` owns the row's markup, not the range's data.
- A new `RangeChips` component (own file, own test) renders the four
  chips, the histogram and the label from `{days, index, nowMs}` props. It
  reads no signal itself — `Board.tsx` hands it the values, as it does
  for every other child.
- The health tallies (`GREEN`/`BLOCKED`/`NO RUN-END`) and the project
  dropdown counts stay **window-scoped**: they describe what is on screen.
  The `n of N` label is what tells the operator the archive is bigger.

**Not in this ticket, but named because the measurements above found them**
(each is its own ticket; do not fold them in here):

1. `handleGrid`/`handleRun` await the usage probe before returning HTML —
   ~3 s cold. The shell should not block on it.
2. Per-run deltas (or at least "don't re-send unchanged runs") on `/events`
   — the live-tick cost is per tab and proportional to the whole window.

## Requirements

- [ ] `GET /events?since=<epochMs>` returns only runs whose
      `runStartEpoch(runId) >= since`; runs outside the window are **never
      read** (assert with an injected counting reader / a `runs/` fixture
      whose out-of-window journal is unreadable and must not raise a notice).
- [ ] `GET /events` with no `since` behaves exactly as today (existing
      `server.test.ts` cases unchanged and green).
- [ ] A run id with no parseable epoch is included without `since`, excluded
      with it.
- [ ] Every frame carries `index: {total, epochs}` derived from the
      directory listing, newest first, parseable ids only.
- [ ] The header row is `[1D][3D][7D][ALL] histogram label │ project ▾ │
      health chips`, chips using the existing `.chip` rules; exactly one
      chip is `on`.
- [ ] `?days=1|3|7|all` in the URL selects the window; absent = `1`;
      changing it updates the URL (`replaceState`) and re-opens the stream
      with the matching `since`; a reload lands on the same window.
- [ ] The histogram lights the in-window days and the label reads
      `since <Mon D> · <in> of <total>` / `all <total> runs`, from
      `index` alone — no second request.
- [ ] Collapsed sections survive a window change (existing `Section` key
      contract).
- [ ] The `Board`/`Header`/`RangeChips` components read no clock and no
      signal themselves — values arrive as props (`src/web/ui/README.md`).
- [ ] Measure and state in the PR body: first-frame server ms and wire KB
      for `1d` vs `all` on the operator's `runs/`, before and after.

## Verify

- [ ] Red test first, `test/web/server.test.ts`: with three fixture runs at
      epochs *now−1h*, *now−2d*, *now−9d* and `since=now−24h`, the first
      `/events` frame carries exactly one view and the reader was never
      called for the other two. RED today — `since` is ignored.
- [ ] Red test: the frame's `index.epochs` lists all three, newest first,
      `index.total === 3`. RED today — no such field.
- [ ] Red test, `test/web/ui/Header.test.tsx`: the range slot renders as
      the first child of `.gh-r`, before `details.menu`. RED today — no slot.
- [ ] Red test, `test/web/ui/RangeChips.test.tsx`: `days=3` → the `3D`
      chip alone is `on`, days 0–2 lit in the histogram, label
      `since … · <in> of <total>`; `days=all` → `ALL` on, every bar lit,
      label `all <total> runs`.
- [ ] Red test, `test/web/ui/live.test.ts` (or `entry`'s edge helper):
      `?days=7` → the events URL carries `since = connectMs − 7·86 400 000`;
      absent → 1 day; `all` → no `since`.
- [ ] Regression: an id without an epoch — present without `since`, absent
      with it.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] Open `adw web` on the operator's `runs/` (97 runs, ≥1 live): the
      first paint shows the last day only; click `7D`, then `ALL`, then
      `1D`; the label and tallies follow; a collapsed section stays
      collapsed across the swaps. Then open `.proto/lazy` side by side and
      confirm the header reads the same.

## Out of scope

- The usage-probe await in `handleGrid`/`handleRun` (separate ticket).
- Per-run deltas / not re-sending unchanged runs on `/events` (separate
  ticket). This ticket shrinks that cost by the window ratio; it does not
  remove it.
- The run screen (`/run`, `/run-events`) — one run, already windowed by
  construction.
- Any change to `card.ts`, `projection.ts`, `board.ts`, `metrics.ts`
  derivation (R5 of `specs/adw-v1.11-render-architecture.md`). The
  filter happens **before** derivation; the derivation is untouched.
- Promoting prototype code. `.proto/lazy/` was written under prototype
  constraints (no tests, a second Preact root glued into the header's
  DOM, a separate `/index.json`). It is the **picture**, not the patch.
  Delete `.proto/lazy/` when this ticket is `done`.
