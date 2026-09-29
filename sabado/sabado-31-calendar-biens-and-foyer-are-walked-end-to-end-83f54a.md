---
id: sabado-31-calendar-biens-and-foyer-are-walked-end-to-end-83f54a
type: feat
status: done
priority: 1
created: 2026-09-28
caps: {minutes: 240, turns: 700, stallMinutes: 25}
depends: [sabado-29-every-pr-walks-a-real-browser-through-a-real-backend-a98ee9]
attempts: [{"runId":"sabado-31-calendar-biens-and-foyer-are-walked-end-to-end-83f54a-1790637176552","branch":"adw/sabado-31-calendar-biens-and-foyer-are-walked-end-to-end-83f54a","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-31-calendar-biens-and-foyer-are-walked-end-to-end-83f54a-1790637176552/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1315","provider":"claude","model":"claude-sonnet-5-5"}]
---
# test(e2e): calendar, biens and foyer are walked end to end, so the pages under active development stop shipping unprotected

> **Why these three, and why P1 (operator, 2026-09-28).** The audit's eight
> journeys were drawn on 2026-09-19. The **72 commits that landed on `main`
> since** moved somewhere else: `ds` ×12, `chat` ×4, **`agenda` ×4**, `foyer` ×3,
> **`biens`/`creation` ×4**, `vault` ×2, `calendar` ×2, `auth` ×2 — from a second
> contributor, continuously, and a refactor is about to start on top. The eight
> journeys cover `vault`, `calendar`'s feed and `auth`; they do **not** cover the
> calendar/agenda **screen**, `biens` creation, or the `foyer` two-step. Those
> three are the pages being changed right now, so they are the ones a PR must
> walk before it merges.
>
> **Deliberately not here.** `ds` dominates the churn but is already gated by
> `ds:check` plus the TSM-01 ratchets — an e2e journey adds nothing there.
> `chat` has no AI provider in CI, the same reason `sabado-26` excluded « Parler
> à Sabado ».

## Requirements

Each journey is its own spec under `frontend/e2e/journeys/`, uses the
`sabado-28` account fixture, and **never calls `page.route`**. Each asserts
against the API as well as the DOM: a screen that renders while persisting
nothing is the failure this suite is for.

- [ ] **R1 — the calendar screen, not just its feed.**
      `10-calendar-screen.spec.ts`. Create an event from the grid → it appears
      in month view → **the event panel opens beside the event it reads**
      (`#1295`) → edit it → the change persists across a reload. Assert
      `GET /calendar` agrees with the grid. Journey 7 (`sabado-30` R5) covers
      the ICS round trip and is not repeated.
- [ ] **R2 — where an event came from is visible.**
      `11-calendar-provenance.spec.ts`. An imported event carries its
      provenance pill, and the pill **sits with the title** (`#1293`, `#1299`);
      an imported event belongs to the account and an empty one gets its person
      (`#1291`). These are three landed fixes with no browser test between them.
- [ ] **R3 — a bien is created through its track.**
      `12-biens-creation.spec.ts`. `/biens/nouveau` → pick a track →
      `/biens/nouveau/:track` → fill the creation parameters → the bien exists
      on `/biens` and in `GET /assets` → open `/biens/:id` and the values
      round-trip. Walk **one** track end to end and assert the **track router**
      reaches each of the others — not every track's full form.
- [ ] **R4 — the foyer two-step.** `13-foyer.spec.ts`. `/foyer/nouveau/:track?`
      → the partenaire two-step (`feat(foyer)`, the
      `feat/foyer-partenaire-two-steps` branch) → the member appears on `/foyer`
      and in `GET /children` or the foyer endpoint it belongs to → open
      `/foyer/:id` and the values round-trip. **Going back one step must not
      lose what step 1 collected** — that is the failure a two-step form has.
- [ ] **R5 — every authenticated route still mounts.**
      `14-authenticated-routes.spec.ts`. Logged in as the fixture account, visit
      **each of the 26 authenticated routes** `App.tsx` declares and assert
      `#root` is non-empty with **no `pageerror`** — the same three listeners
      `bundle/boot.spec.ts` uses, against the **real** API instead of a stub.
      Routes with a `:param` use a record the journey itself created. This one
      test is the widest coverage-per-line in the suite: **26 routes that no
      browser visits today.**
- [ ] **R5b — `App.tsx` exports its route table, and that is production code.**
      R5 must not hardcode a route array: one goes stale the week a route is
      added, which is the failure mode this ticket exists to prevent. But
      `frontend/src/App.tsx` **today exports only `export default function
      App()`** — there is no route table to import, and the routes live inside
      JSX behind `lazy()` wrappers. So R5 requires **extracting the route list
      into an exported `const`** (e.g. `export const ROUTES` carrying
      `{ path, lazy, authenticated }`) that `App()` then renders from, and which
      the spec imports.
      **This is a production-code refactor inside a test ticket — it is named
      here deliberately rather than discovered mid-build.** It must be
      behaviour-preserving: the rendered route tree is identical, every existing
      vitest case stays green untouched, and `bundle/boot.spec.ts`'s
      `lazyChunks.length > 24` assertion still holds (it counts `lazy()` pages,
      so the extraction must not collapse them into the entry chunk). If the
      extraction turns out not to be behaviour-preserving, **stop and say so**
      rather than adjusting the assertion.
- [ ] **R6 — the budget is reported, not silently paid.** Record the `e2e` job's
      measured wall-clock before and after this ticket in the PR body. If the
      job passes 8 minutes (`sabado-29` R8), **say so and change nothing** —
      whether to split the job, shard it, or pay it is the operator's call.

## Files

`frontend/e2e/journeys/10-calendar-screen.spec.ts` ·
`11-calendar-provenance.spec.ts` · `12-biens-creation.spec.ts` ·
`13-foyer.spec.ts` · `14-authenticated-routes.spec.ts` (all new) ·
a helper under `frontend/e2e/fixtures/` if R5's route derivation needs one.

## Verify

- [ ] All five green in the `e2e` job; `grep -rn "page.route"
      frontend/e2e/journeys/` → no match.
- [ ] **R5 covers every route:** the count of routes the spec visits equals the
      length of `App.tsx`'s exported authenticated-route list, asserted in the
      test itself — so adding a route without covering it goes red.
- [ ] **R5b is behaviour-preserving:** every pre-existing vitest case green with
      no edit, `bundle/boot.spec.ts` green including its `> 24` lazy-chunk count,
      and the built `dist/assets` chunk count unchanged by the extraction.
- [ ] **Red proof, R1:** break the event-panel positioning (revert `#1295`'s
      one positioning rule locally) → `10-calendar-screen.spec.ts` goes red
      naming the panel. Record it in the PR body.
- [ ] **Red proof, R4:** the back-step case fails against a build where step 1's
      state is dropped.
- [ ] The measured job duration, before and after, is in the PR body.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `test-e2e-boot` · `just test` — green.

## Out of scope

`ds` journeys (`ds:check` + the ratchets already gate it) · « Parler à Sabado »
and `chat` (no AI provider in CI) · `/budget`, `/contacts`, `/settings` and
`/sabado` beyond R5's mount check · visual regression · Firefox and WebKit.
