---
id: adw-bug-29-every-sse-stream-dies-every-ten-seconds-4b0e17
type: bug
status: queued
priority: 1
created: 2026-09-28
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# "⚠ connection lost" every 15 seconds, on every screen, forever

> **Evidence, 2026-09-28.** The operator reports the stale banner keeps
> appearing. It is not a false positive and it is not intermittent: **every
> SSE stream `adw web` serves is killed by the server ~10 seconds after its
> last byte**, on all three routes.
>
> `Bun.serve` is called with no `idleTimeout` (`server.ts:329`), so Bun's
> default of 10s applies. None of the three SSE routes sends anything on a
> quiet tick — `handleEvents`/`handleRunEvents` enqueue only when the
> serialized body differs from `lastSent` (the honest push gate), and there
> is no keep-alive comment frame anywhere in the module. A finished run, or
> a board with nothing live on it, therefore goes silent immediately after
> its first frame and is cut off 10 seconds later.
>
> Measured against a real server on the real `runs/`:
>
> | route | held | curl |
> |---|---|---|
> | `GET /run-events` | 12.06s | `rc=18` (partial file — server closed mid-body) |
> | `GET /events?since=0` | 12.07s | closed |
> | `GET /backlog-events` | 12.06s | `rc=18` |
>
> A minimal `Bun.serve` repro isolates the cause and Bun names it itself:
> a default server cut an idle stream at **12.0s** and printed
> `warn: Bun.serve() timed out a request after 10 seconds. Pass idleTimeout
> to configure.`; the same server with `idleTimeout: 0` held the same idle
> stream for the full 40s cap. (The warning does not appear in `adw web`'s
> own stdout, so the operator never saw it.)
>
> Instrumented `EventSource` in Chrome, against a finished run's
> `/run-events`:
>
> ```
> 0.0s  open          0.0s  message 121038b
> 10.7s ERROR readyState=0      13.7s open   13.7s message 121038b
> 25.7s ERROR readyState=0      28.7s open   28.7s message 121038b
> ```
>
> A ~15s cycle, forever. `onerror` adds `body.stale` (`live.ts`), `onopen`
> removes it, so **the banner is visible ~3s out of every 15 on every open
> tab**. Independently confirmed by sampling `document.body.classList` on
> the board: `1.0s stale=false | 10.0s stale=true | 14.0s stale=false`.
>
> Two consequences beyond the banner: the browser re-fetches the **entire**
> payload on every reconnect (121 KB for that run — ~8 KB/s per open tab,
> indefinitely), and the server pays a full journal-tail + `composeRunView`
> + `buildDrawers` + sidecar re-hash for each one, because `lastSent` and
> `promptMemo` are per-connection and a reconnect is a new connection. The
> `adw-fe-24` "each sidecar read and hashed at most once per SSE connection"
> guarantee is intact but now means "once per 15 seconds".
>
> This is NOT `adw-bug-28`. That one is a blank run screen (a throw in
> `resolvePrompt` on a hopped node); this one is the stale banner, and it
> fires on runs that render perfectly.

## Root cause

Bun's default 10s `idleTimeout` versus a protocol that is deliberately silent
between events. The honest push gate (`adw-fe-08` R5/R6) is correct and should
not change — the missing piece is that SSE needs a keep-alive to distinguish
"nothing happened" from "nothing is there".

## Requirements

- [ ] **R1 — red first.** A `handleRunEvents`/`handleEvents` test asserting
      that a connection with no journal change emits a keep-alive within the
      idle window. Fails on `main`, where a quiet tick emits nothing at all.
- [ ] **R2 — a keep-alive comment frame.** On a tick that pushes no data,
      write an SSE comment (`:\n\n` or `: ping\n\n`) at a cadence safely
      under the configured `idleTimeout`. A comment is ignored by
      `EventSource` by spec, so `onmessage`/`applyRunEvent` must not fire and
      `runData`/`boardData` must not change — assert that, not just the bytes.
      Applies to all three routes (`/events`, `/run-events`,
      `/backlog-events`); they share the defect, so they share the fix.
- [ ] **R3 — set `idleTimeout` explicitly**, rather than inheriting a default
      that silently changes between Bun versions. Pick it against R2's
      cadence and say so in the comment. Prefer a finite value with a
      heartbeat under it over `idleTimeout: 0`: a finite timeout still
      reclaims a genuinely leaked connection, and `0` would make the stale
      banner unable to mean anything in the other direction.
- [ ] **R4 — the banner must still mean something.** Stop the real server and
      assert the client does show `stale` and keeps showing it. The current
      banner is a false alarm; the fix must not make it a dead indicator.
- [ ] **R5 — no reconnect storm left.** With R2/R3 in place, hold a real
      `/run-events` connection open for ≥60s against a finished run and
      assert exactly ONE data frame was sent. Today it is one per ~15s.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test` green.
- `curl -sN --max-time 60 "http://127.0.0.1:<port>/run-events?id=<finished-run>&token=<t>"`
  runs the full 60s and exits `rc=28` (curl's own timeout), never `rc=18`.
- Open a run screen in a browser for two minutes: no banner, ever.

## Note

`GET /backlog` has no HTML route on the server (`server.ts` serves `/`,
`/usage.json`, `/bundle.js`, `/events`, `/run`, `/run-events`,
`/backlog.json`, `/backlog-events`). The Backlog tab works via `pushState`,
so a **reload** on `/backlog` — or a bookmarked link — returns 404. Seen
while tracing this bug, not part of it; file separately if it is real.
