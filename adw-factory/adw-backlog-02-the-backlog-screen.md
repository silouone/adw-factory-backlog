---
id: adw-backlog-02-the-backlog-screen
type: feat
status: in-review
priority: 1
created: 2026-09-19
review: true
caps: {minutes: 150, turns: 700}
depends: [adw-backlog-01-the-backlog-projection, adw-render-01-foundation]
attempts: [{"runId":"adw-backlog-02-the-backlog-screen-1790524715037","branch":"adw/adw-backlog-02-the-backlog-screen","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-backlog-02-the-backlog-screen-1790524715037/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/131","provider":"claude","model":"sonnet"}]
---
# The backlog screen — a third tab, born as components

> **Spec authority:** `specs/adw-v1.14-backlog-tab.md` §3 **D1, D6, D8,
> D11** and §8. Conventions: `src/web/ui/README.md` and the pattern
> `adw-render-01-foundation` fixed — components in `src/web/ui/`, tests in
> `test/web/ui/` (**not** colocated: `bunfig.toml` roots tests at `test/`;
> a colocated test is silently never run — check the test count goes up).
> `review: true`: this is judgement-shaped work (layout, legibility).
>
> **Does not depend on P1–P3.** The screen ports no legacy code; it consumes
> `adw-backlog-01`'s JSON routes directly. It is the second proof of P0.

## What is being built

`GET /backlog`: a shell that loads `/bundle.js` and mounts `<Backlog/>`,
which subscribes to `/backlog-events`, holds the `BacklogView` in a signal,
and renders D6/D11. Everything is a pure function of props; the one `now`
signal at the root is the only clock reader (`elapsed`/age are `computed()`
over it, from `lastAt` timestamps).

## Requirements

- [ ] **R1 — state-major sections, project filter (D6).** Sections in order
      `blocked` → `waiting` (hard first) → `ready` → `running` → `in-flight`;
      a project dropdown with the board's labels and gesture; a per-project
      count strip (`clens · 1 running · 3 ready · 1 waiting · 2 closed`);
      store-less targets listed as `no ticket store`.
- [ ] **R2 — the row (D11).** State glyph · short id (full on hover) · title
      · lane · priority · dep chips coloured hard/soft · `attempts · last
      outcome · age` · PR link when present · `review: off` marker · copy
      button (copies the id). `stale` rows carry a visible marker.
- [ ] **R3 — closed toggle (D5).** Off by default; `done` and `rejected`
      with distinct glyphs; counts always visible in the strip.
- [ ] **R4 — the hero.** The top `ready` row of the active filter; its copy
      button copies `TARGET=<project> just run <id>`.
- [ ] **R5 — the detail panel (D11).** Clicking a row opens a side panel on
      the tab: every frontmatter field, the body rendered as markdown
      (reuse `render-run.ts`'s `mdToHtml` by lifting it into a pure module
      the component imports — do not write a second renderer), the
      `attempts:` ledger with each run linked to `/run?id=`. Open state,
      selected row and panel scroll are signals; they survive every push.
- [ ] **R6 — nav (D8).** A `Board · Backlog` strip in the component, and
      one `<a href="/backlog?token=…">` in the legacy board header
      (`render.ts`'s `renderBoardHeader`). P2 removes the legacy anchor.
- [ ] **R7 — no derivation in components.** If the screen seems to need a
      field the view-model lacks, it belongs in `backlog.ts` with its own
      red test — not computed in TSX.
- [ ] **R8 — read-only.** No form, no fetch other than the two `GET`s.

## Verify

- [ ] Red test first (Art. I): `<Backlog/>` given a fixture view renders one
      row per row, in D6 order, with hard and soft dep chips distinguishable
      by class. RED today (§8 criterion 2, on screen).
- [ ] Red test: a closed row is not rendered until the toggle is on; the
      strip's closed count is rendered regardless.
- [ ] Red test: the panel stays open with the same selected id across a
      view-model update (§8 criterion 5).
- [ ] Red test: the hero's copy payload is the full dispatch line.
- [ ] Red test: a `hasStore: false` project renders a `no ticket store` line.
- [ ] Manual, live: open `/backlog` beside a running ticket; the row flips
      to `running` without reload; collapse a section, wait 10 ticks, it is
      still collapsed.
- [ ] Manual: `just next` per target agrees with the `ready` section per
      project (§8 criterion 3).
- [ ] `bun test` count goes **up** by the number of new component tests.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

- Any change to the board or run screen beyond the one nav anchor.
- Per-row spend; GitHub state; dispatch from the browser.
- Redesigning `mdToHtml` — lift it, do not rewrite it.
