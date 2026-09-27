---
id: adw-bug-28-a-hopped-node-blanks-the-run-screen-7c31a9
type: bug
status: queued
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A hopped node's run screen renders nothing at all

> **Evidence, 2026-09-27.** The operator clicked a run on the board and got a
> page holding only `← all runs`. The browser console showed nothing but
> Chrome-extension noise and a favicon `401` — because the failure is
> server-side and swallowed.
>
> Reproduced by enumerating every banked run: 130 runs probed against
> `GET /run-events`, **7 of them never push a single SSE frame**
> (`adw-backlog-01-…-1790517794046`, `adw-cost-04-…-1790473998753`,
> `adw-perf-06-…-1790476027269`, `adw-perf-06-…-1790486509641`,
> `adw-skills-01-…-1790522958340`, `adw-usage-04-…-1790473626224`,
> `adw-usage-04-…-1790479260068`). All seven throw on the same line:
>
> ```
> error: run "…" ticket "…": node "plan" has more than one prompt record
>        matching this occurrence's window
>   at resolvePrompt   (src/web/drawer.ts:245)
>   at buildDrawerView (src/web/drawer.ts:174)
>   at buildDrawers    (src/web/run-view.ts:180)
> ```
>
> All seven are **hopped** runs. `resolvePrompt`'s own comment
> (`drawer.ts:221`) states the invariant it relies on: *"more than one match
> is structurally impossible under `makePromptSink`'s per-call unique
> sequence."* That was true until the turn-cap/context hop landed
> (`adw-bug-26`, `adw-bug-27`, `adw-cost-06`, all merged 2026-09-27). A hop
> opens a **fresh session inside the same node occurrence**, and each session
> writes its own `prompt` record. skills-01's `plan` occurrence
> (`node-start 1790524224912` → `node-end 1790525016107`) contains **three**
> prompt sidecars (`0001-plan.txt`, `0002-plan.txt`, `0003-plan.txt`) and two
> `turn-cap-hop` events.
>
> Per the 2026-09-27 orchestration log D38, hops are now **the common case**
> ("most builds hop at cap 60"). This is not 7 stale journals: every future
> run that hops renders a blank run screen.

## Root cause, in two parts

1. **`resolvePrompt` throws on a legitimate shape.** One node occurrence, N
   sessions, N prompt records, one `prompt` slot in `DrawerView`.
2. **`handleRunEvents`'s tick swallows it** (`server.ts:1016`, bare `catch {}`).
   The throw is permanent, so the stream stays open and *never* pushes a first
   frame. `runData` stays `undefined`, `<Run/>` renders the crumb and stops.
   No console error, no `stale` class, no notice — indistinguishable from
   "still loading". This is why it shipped unnoticed.

## Amendment required before implementing

**Stop and get the spec answer first.** There is no specified product for
"which prompt does a hopped node's drawer show". First session, last session,
all N, or a distinct non-throwing state are four different screens, and
`drawer.ts`'s own N5 rule (*a wrong claim on screen is worse than no claim*)
forbids silently rendering session #1 as though it were the whole occurrence.
`adw-usage-05` already answered the sibling question for **tokens** (a
multi-session node reports only its last session); prompts need the same
treatment in `adw-v1.11-render-architecture.md`. Propose the amendment, get
it approved, then write the red test.

## Requirements

- [ ] **R1 — red first.** A `buildDrawers` test over a node occurrence holding
      two `prompt` records (the real skills-01 shape: `node-start`, prompt,
      `turn-cap-hop`, prompt, `node-end`). It throws on `main`.
- [ ] **R2 — a second red at the route.** A `handleRunEvents` test proving the
      SSE stream pushes a first frame for that same journal. This is the
      requirement that actually matches the operator's symptom; R1 alone
      could be satisfied while the screen stays blank.
- [ ] **R3 — the drawer renders every session of a hopped occurrence**, per
      the approved amendment. Zero matches still folds to `not-persisted`.
      A single-prompt occurrence renders byte-identically to today (assert it:
      126 of 130 banked runs must not move).
- [ ] **R4 — the `PromptMemo` contract survives.** Each sidecar is still read
      and hashed at most once per SSE connection (`adw-fe-24`). N prompts per
      occurrence must not become N re-hashes per tick.
- [ ] **R5 — a swallowed compose failure is never a silent blank screen.**
      `handleRunEvents` currently cannot tell the client anything went wrong.
      Give it the board's own `projectionNotices` posture, or an explicit
      error frame the run screen renders. Fail-fast (Art. IX) that ends in a
      blank page is not fail-fast.
- [ ] **R6 — no sibling assumption is left standing.** `buildGanttView`
      (`gantt.ts:168`) makes a similar "structurally impossible" claim about
      node nesting; it was verified unaffected by hops on all 130 banked runs
      (every one of the 7 failures reached `composeRunView` fine and died in
      `buildDrawers`). Re-verify and record the result — do not widen scope.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test` green.
- The enumeration that found this, re-run: probe `GET /run-events?id=<id>` for
  every directory in `runs/` and assert **0 runs push 0 frames**.
- Load `/run?id=adw-skills-01-a-target-loads-its-own-skills-eca3a0-1790522958340`
  in a browser and see the Gantt, the plan drawer, and its prompts.

## Do not dispatch yet

Two factory runs were in flight when this was filed (`adw-skills-01`,
`adw-backlog-02`), and skills-01's own run is one of the seven blank screens.
Per the firing protocol, dispatch after they reach a terminal outcome.
