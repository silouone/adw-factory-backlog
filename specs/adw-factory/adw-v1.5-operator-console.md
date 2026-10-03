# Amendment v1.5 — from observability surface to operator console

> **Status:** proposed 2026-09-14. **D1 answered by the operator 2026-09-18**
> (see §3 D1); D3 still open.
> **Amends:** `adw-v1.2-live-view.md` decisions 11 and 2's consequences (zero
> new runtime dependencies; two screens only). **Binding:**
> `constitution.md` — unchanged. **Depends on:** `adw-fe-12-wire-the-run-screen`
> (there is no point restyling screens that are not yet reachable).
> **Implemented by:** `tickets/adw-console-01-operator-console.md`.

## 1. Trigger

v1.2 §Solution scoped the view deliberately narrowly, and decision 11 said why:

> **Zero new dependencies.** Two screens of positioned rectangles and a drawer
> do not need a framework, and every dependency added here is a dependency in
> the factory.

That was the right call for v1.2, whose purpose was to settle two Art. VI debts
— the heartbeat and prompt persistence — and to prove the journal could carry a
view at all. Both are done.

The trigger for revisiting it is operator experience, stated 2026-09-14 against
a live `adw web`:

> *"I need to be able to enter each ticket, have filter per project, a real
> smooth shiny FE, easy to navigate and with good UI/UX, a live capability to
> track ongoing progress on a task preventing me from always wondering where we
> are, clear visibility into dependencies, and phases & model running the task
> and ram/cpu."*

Four of those v1.2 already covers once `fe-08` and `fe-12` land — entering a
run, phases, model, live progress. **Three it never scoped at all**, and one of
them cannot be served from the data the factory records today.

## 2. What this amendment adds

| want | v1.2 | v1.5 |
|---|---|---|
| enter a run, phases, model, live | covered by `fe-08` + `fe-12` | — |
| **filter / group by target** | not scoped | **yes** |
| **dependency visibility across tickets** | not scoped | **yes** |
| **navigable, styled UI** | explicitly excluded (decision 11) | **reverses it** |
| **host RAM / CPU per run** | not scoped, **and not recorded** | **needs new instrumentation** |

## 3. The three real decisions

**D1 — does the factory take a frontend dependency?** This is the whole
amendment. v1.2's "every dependency added here is a dependency in the factory"
argument does not become wrong because the operator wants nicer screens; it
becomes a **trade** the operator can now price, having used the alternative.
Options, in ascending cost: hand-written CSS and a little vanilla JS (no
dependency, real limits); one small styling dependency; a full framework and a
build step in a repo that currently has none.

> ### D1 — ANSWERED 2026-09-18: **yes, Preact + `@preact/signals`.**
>
> **Decided by the operator**, reversing v1.2 decision 11. Stated rationale,
> verbatim: *"1) I wanna try it 2) it's lightweight 3) I don't think we need
> the full React for such a small project for now."*
>
> **The forcing function was not aesthetics.** It was the defect investigated
> 2026-09-18 (`ai_docs/2026-09-18-web-fe-render-thrash.md`): both screens
> replace their entire DOM ~1×/s, destroying every piece of operator state —
> `<details>` expansion, scroll position, text selection, keyboard focus. The
> shipped mitigation strategy (hoist one widget's flag into a closure, re-apply
> after the swap — `adw-usage-01` PR #61, then again for the usage chip, then
> again for the dock scope) is O(one patch per widget) and structurally cannot
> reach scroll, selection or focus. **This decision is what makes that class of
> defect uncloseable-by-patching go away.**
>
> **What was measured before deciding**
> (`ai_docs/2026-09-18-web-fe-framework-decision.md`):
>
> - **The migration is a view-layer replacement, not a rewrite.** 3,446 of
>   `src/web/`'s 7,384 lines emit zero HTML and survive untouched; 7,602 of
>   `test/web/`'s 12,536 lines survive. Only `render.ts` + `render-run.ts`
>   (2,448 lines) are replaced.
> - **Svelte and Vue were eliminated by the gates, not by taste.**
>   `bunx tsc --noEmit` reports real errors for `.tsx` and emits **zero
>   diagnostics** for `.svelte`/`.vue` in the same directory. Since the factory
>   dispatches against itself, an SFC stack would ship agent-authored component
>   logic **untypechecked** unless `svelte-check`/`vue-tsc` is added as a second
>   toolchain. Solid was eliminated for requiring babel.
> - **Verified by spike:** 3 runtime packages; `bun build` produces **8.1 KB
>   gzipped in 7 ms** with no new build tooling; `exactOptionalPropertyTypes`
>   enforces through JSX props (`TS2375`); and the Article I red→green loop runs
>   on the **existing** `bun test` via `@testing-library/preact` + `happy-dom`.
>
> **Constraints this decision carries** — binding on every ticket that
> implements it:
>
> 1. **Derivation stays server-side.** The SSE wire carries serialized
>    `RunView`/`CardSummary` **JSON**, never HTML and never raw journal records.
>    This is what protects the 3,446 surviving lines; reversing it later is
>    expensive.
> 2. **`elapsed` becomes a client-side ticker over `startedAt`.** The wall clock
>    must not leave the browser, so the server pushes only when journal bytes
>    actually arrive. This structurally retires defect D2 of the thrash report.
> 3. **No second toolchain.** `bun run lint && bunx tsc --noEmit && bun test`
>    stays the whole gate. `bun build` is the only addition, and only for the
>    client bundle.
> 4. **The console stays read-only** (§4 non-goals, unchanged).
>
> **Still open:** D3 below. D1 does not unblock
> `tickets/adw-console-01-operator-console.md` on its own — that ticket also
> needs the ticket-store boundary settled, and keeps `status: blocked`.

**D2 — filtering and grouping need no new data.** `run-start.target` has been
journaled since `adw-fe-01`. This is projection and UI work only, and is the
cheapest item here.

**D3 — dependency visibility crosses a boundary.** `depends:` lives in ticket
frontmatter, not in any journal. A dependency view means the web layer reads the
**ticket store**, which today only `intake/` touches — and which
`adw-v1.4-ticket-store.md` proposes to move out of the repo entirely. These two
amendments interact; v1.4 should be settled first or at least read alongside.

**RAM/CPU is called out separately because it is not a view problem.** Nothing
in the factory samples host resources. Serving it means the engine sampling and
journaling them per node — new instrumentation in the hottest path, for a
number whose value is unproven. Measured 2026-09-14: a whole run's own
processes are ~400 MB against a machine already carrying ~24 GB of editor and
browser, so the factory's own footprint is close to noise. **Recommend
deferring** until a real question needs it.

## 4. Non-goals

Unchanged from v1.2: no router agent, no watcher daemon, no scheduled runs, no
queue draining, and **no mutating route** — the console stays read-only. An
operator console that can dispatch is a different amendment with a different
risk profile.

## 5. Requires an operator decision

D1 changes a stated architectural constraint. Per Article IV the operator sits
at the two ends; this is one of them. Nothing in this amendment is buildable
until D1 is answered, which is why its ticket ships `status: blocked`.

**D1 was answered 2026-09-18 — see §3.** The frontend dependency is taken:
Preact + `@preact/signals`, under the four constraints listed there. **D3
remains open**, so `adw-console-01` keeps `status: blocked`; what D1 unblocks
is the *render-architecture* work (the view-layer replacement), which is
scoped separately because it is an enabler for this amendment rather than a
part of it.
