---
id: adw-fe-16-the-run-screen-does-not-move
type: feat
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-fe-16-the-run-screen-does-not-move-1789457835082","branch":"adw/adw-fe-16-the-run-screen-does-not-move","workspace":"/Users/silouane/adw-factory/runs/adw-fe-16-the-run-screen-does-not-move-1789457835082/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/49","provider":"claude","model":"sonnet"}]
---
# The run screen is static — the one screen you open to watch a run needs F5

The board live-updates over SSE. The run screen does not. An operator who
drills into a run to watch it — the exact motion `adw-v1.2` was written for —
gets a frozen snapshot and has to reload by hand.

> *"a live capability… preventing me from always wondering where we are"*
> — `specs/adw-v1.2-live-view.md`, the trigger quote

Reported by the operator 2026-09-15 with two screenshots of the same live run
50s apart: identical, until a manual reload.

## The evidence

- `grep -n EventSource src/web/*.ts` returns exactly one hit, `render.ts:602`
  — the board. The run page's script (`render-run.ts:294-307`) only wires the
  dock's click/Escape handlers.
- `server.ts:139` routes `/run` to `handleRun`, which returns one static
  `renderRunPage` string. There is no `/run-events`.

## Three things are wrong together, and fixing one without the others is worse

### 1. No live refresh
- [ ] A `/run-events?id=` SSE route, mirroring `handleEvents`'s existing
      per-connection `RunTailState` (byte offset + accumulated records) —
      `tailer.ts` already does the hard part and needs no change.
- [ ] Push the re-rendered `.lanes` fragment (and the header chips), not the
      whole document. `renderRunPage` is pure; factor the lane rows into a
      `renderLanes(view, …)` the route and the page both call (Art. VIII).
- [ ] Register `onerror`: on a dropped stream, show a stale banner and strip
      the live treatment. **Same defect as the board carries today** —
      `render.ts:602` has `onmessage` only, so a dead stream leaves a beating
      beacon forever. Fix both, in one place if possible.

### 2. `dur` reads `—` for a live run's entire life
`projection.ts:57-59` sets `durationMs` only when a `run-end` exists. So the
header chip and the board's time KPI show `—` for exactly as long as the run
is interesting.

- [ ] While `state !== "finished"`, derive elapsed from the run's own start
      and label it **`elapsed`**, not `dur`. The two are different claims and
      must not share a label.
- [ ] `runId` is `<ticketId>-<epochMs>`; the epoch suffix is already parsed by
      `projection.ts`'s last-resort ticketId derivation. Prefer the first
      journal record's timestamp, and fall back to the suffix.

### 3. A fresh run renders a red failure pill reading "unknown"
`deriveRunState` (`projection.ts:74-92`) returns `"unknown"` for a run's first
~15s, before the first heartbeat. `render-run.ts:253-258` maps anything that
is not `running`/`green` to `.st.bad`.

- [ ] A third status class — neutral `--dim`, not `--bad` — for
      `state === "unknown"` with no `run-end`. "We have not heard from it yet"
      is not a failure, and a red pill on every run's first 15 seconds trains
      the operator to ignore red.

## Verify

- [ ] A live run's run screen advances without a reload; blocks grow, new
      nodes appear, the header chips update.
- [ ] Killing the server mid-run leaves a visible stale banner, not a frozen
      page pretending to be live. Same on the board.
- [ ] A run in its first 15s shows a neutral pill and an `elapsed` figure that
      is not `—`.
- [ ] The lane-row renderer has one home; grep proves the route and the page
      call the same function.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

The ghost chain (`adw-fe-17`). Deep-linkable filter/selection state. Anything
in `adw-fe-15`.
