---
id: adw-render-02-the-wire-carries-view-models
type: feat
status: done
priority: 1
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: [adw-render-01-foundation, adw-fe-24-the-live-tick-erases-every-prompt-from-the-panel]
attempts: [{"runId":"adw-render-02-the-wire-carries-view-models-1789807711430","branch":"adw/adw-render-02-the-wire-carries-view-models","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-render-02-the-wire-carries-view-models-1789807711430/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# P1 — the SSE routes push view-models, and the wall clock stops crossing the wire

> **Spec authority:** `specs/adw-v1.11-render-architecture.md` §2.1, §2.2,
> §4 **P1** and §6 **D1**.
>
> **Ordering is now enforced by the engine, not by this paragraph
> (2026-09-19).** `adw-render-01-foundation` was rescoped from `type: manual`
> to `feat`, so it parses, so it appears in the `allTickets` set
> `unmetDependsRefusal` (`src/cli.ts:630`) checks — and it is now listed under
> `depends:`. Dispatching this ticket before P0 is `done` is refused with
> `EXIT_REFUSED`, naming P0 and its current status.
>
> The previous note said the status field was the only thing holding the line,
> because a dep naming a non-parsing ticket is **silently treated as met**
> (`cli.ts:627`, deliberate — a typo must not masquerade as an unmet dep).
> That was true while P0 was `manual`. It is not true now, so this ticket
> returns to `queued` and the guard does the work.
>
> `depends:` also carries `adw-fe-24` (merged, PR #79) — see below for why
> that ordering matters.

## Why `adw-fe-24` must land first

`adw-fe-24` restores the compiled prompts to the live run-screen fragment
(today `handleRunEvents` passes `new Map()` for `drawers`, so a tick renders 0
prompts where the static page rendered 2). Its fix lands in `server.ts` and
`run-view.ts` — modules this migration **keeps** — and it establishes the
per-stream prompt memo. **That work is this ticket's first slice**: a
view-model that omits what the static render included is not a view-model.
Landing P1 first would mean fe-24 was written against a path already deleted.

## The defect being retired

Measured 2026-09-18 (`ai_docs/2026-09-18-web-fe-render-thrash.md`): **11 SSE
pushes in 10 s against 0 new journal bytes**, because `if (body !== lastSent)`
compares strings containing `19m 42s` and `left:29.284%`. The board pushes
302.6 KB per frame; the run screen 15.1 KB.

## Requirements

- [ ] **R1 — the payload is view-model JSON.** `/events` emits the
      `RunView[]` + `CardSummary` map; `/run-events` emits the run's header,
      `GanttView`, up-next and panel data. These are the interfaces
      `src/web/` **already exports and already tests** — reuse them verbatim,
      do not invent a parallel DTO.
- [ ] **R2 — derivation stays server-side (§6 D1).** `projectRun`,
      `buildCardFromRecords` and `composeRunView` keep running on the server.
      The wire carries neither HTML nor raw journal records. Moving
      derivation client-side would drag ~7,600 lines of passing tests with it
      and is a blocker, not an optimisation.
- [ ] **R3 — no wall clock in the payload.** No rendered duration string, no
      computed axis percentage, no `now`-derived float. The payload carries
      **timestamps**; the client derives from them.
- [ ] **R4 — exactly one clock reader on the client.** A single `now` signal
      at the application root, ticking at 1 Hz. `elapsed` and live geometry
      are `computed()` over it and reach components **as props**. No
      component calls `Date.now()` (§7, Art. IX).
- [ ] **R5 — the push gate becomes honest.** The server pushes when the
      serialized view-model changes. With no new journal bytes and no state
      transition, an idle live run pushes **nothing**.
- [ ] **R6 — the liveness check survives.** `projectRun`'s clock-driven
      recompute exists because a hard-killed run stops appending entirely
      (`server.ts`'s own comment): a byte-driven-only poll would show
      "running" forever. **A run that dies must still flip to its dead state
      without new bytes.** Do not delete the clock-driven recompute in
      service of R5 — recompute server-side, push only when the *projection*
      changes.
- [ ] **R7 — both screens still render.** At the end of this ticket the board
      and run screen still produce HTML server-side as they do today; only
      the *transport* has changed, with the client holding the view-model in
      signals alongside. P2/P3 flip the rendering.

## Verify

- [ ] Red test first (Art. I): assert a `/run-events` frame parses to a
      `RunView`-shaped object and contains **no** `%`-suffixed geometry and no
      `m `/`s` duration string. RED today.
- [ ] Red test: with a fixed clock and no new journal bytes, two consecutive
      ticks produce byte-identical payloads and the second is **not** pushed.
      RED today — this is the 11-pushes/0-bytes measurement inverted.
- [ ] Red test for R6: a run whose journal stops appending still transitions
      out of `running`, and that transition **is** pushed.
- [ ] Manual, against a live run: open `/run-events` for ≥10 s with no
      journal growth → **0 pushes**. Compare to the 11 recorded today.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

- Converting either screen to components (P2/P3).
- Any change to what a view *says* — v1.7 and v1.10's contracts are
  preserved verbatim (§"Does NOT amend").
- Redesign (§8).
