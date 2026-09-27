---
id: adw-backlog-03-the-backlog-screen-is-legible-and-usable-4b17e2
type: feat
status: in-review
priority: 1
created: 2026-09-27
review: true
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [adw-backlog-02-the-backlog-screen]
attempts: []
---
# The backlog screen is legible and usable: a design pass on a raw first cut

> **Refined 2026-09-27: the operator picked variant A ("ledger").** Three
> variants were prototyped on real `/backlog.json` data in
> `.proto/backlog/` (A ledger table · B cards · C master/detail triage), plus
> `?v=0` showing today's screen. Operator: "let's go for A, this is good
> enough, build it right there". Built in-session, not dispatched.
> The prototype is the visual reference for the verify step.
> **Built 2026-09-27 → PR #135 (merged); R11 → PR #137** (hand-built, so `attempts:` stays empty).

> **Evidence, operator screenshot 2026-09-27 20:12** (post-#131 + audit fixes
> PR): "backlog is VERY RAW, UI is very bad and then UX as well". adw-v1.14
> scoped the screen as data-first and listed "no redesign" as a non-goal. This
> ticket is that redesign, for the backlog screen only, in the board's language
> (`adw-fe-15`: two screens, one language).

## What is wrong today (from the screenshot)

**Visual**
1. **Project strip is one run-on wrapped line** of all 12 targets, 7 of them
   `no ticket store` noise. It is hard to scan and wider than the viewport.
2. **Buttons are unstyled browser defaults** (white `copy` / `copy run command`)
   on the dark theme, on every row.
3. **Rows are a single run-on text line**: glyph · id · title · lane · priority ·
   attempts · outcome · age · copy. No columns, no alignment, no truncation, and
   long ids push titles around.
4. **Section headers**: a double caret (`▼ ▸blocked`), and the count is glued to
   the label (`blocked6`) with no badge or spacing.
5. **Detail panel**: frontmatter is a tall key/value list with every value on its
   own indented line; the selected id has a garish orange highlight; the markdown
   body keeps source hard-wraps as odd indented fragments; checkboxes render as
   literal `[ ]`; headings are oversized.
6. **Selected row**: a bright yellow id highlight, inconsistent with the board.

**UX**
7. **No project on the row.** Under *All projects* (now sorted across projects),
   short ids like `010-import-codex-autodiscover` don't say they are clens.
8. **Only one filter** (project). No search, no state / lane / priority filter, and
   no filter or selection state in the URL (so nothing is shareable or survives reload).
9. **Row copy copies only the id.** The useful action is the per-row dispatch line
   (`TARGET=<t> just run[-codex] <id>`), which only the hero has.
10. **Why is it stuck?** Blocked rows don't show the block reason (the last
    attempt's outcome and reason); waiting rows depend on dep chips (fixed in the
    audit PR) but don't say "waits on X (blocked)".
11. **Ledger drift is shown, not explained**: `clens-007`/`008` in-flight with 0
    attempts and `clens-011` "stale" appear as normal rows. Drift (a status with no
    run, an in-review with no PR) needs a visible warning with its cause. This is
    the first supervisor-style signal on screen (see
    `ai_docs/2026-09-27-supervisor-toolbox.md`, Tier 1).
12. **No keyboard navigation** (j/k, Enter to open, Esc to close), no focus
    management in the panel.

## Requirements (finalised 2026-09-27, variant A)

- [ ] **R1** The approved design direction is implemented in `src/web/ui/` with
      the board's shared theme and components (styled buttons, chips, badges,
      section headers). No browser-default controls remain.
- [ ] **R2** Rows are a fixed column grid: state glyph · **project** · id (truncated,
      full on hover/copy) · title (ellipsis) · lane · P · deps / why-stuck · last
      outcome + age · actions.
- [ ] **R3** The project strip is compact, one chip per project with counts;
      targets with no store are collapsed behind a single "n targets without a store" toggle.
- [ ] **R4** Filters: project, state, lane, priority, free-text search. The filter
      and selection live in the URL query.
- [ ] **R5** Per-row actions: copy id, copy the dispatch line (provider-aware), open
      the run for the last attempt.
- [ ] **R6** Blocked rows show the last attempt's reason (one line, truncated,
      full in the panel); waiting rows name the unmet dependency and its state.
- [ ] **R7** Drift rows are flagged with the reason ("in-review, no PR recorded",
      "in-progress, no live run"), with the derivation server-side (v1.14 R7:
      no derivation in components).
- [ ] **R8** The panel: a compact frontmatter grid, markdown that reflows source
      hard-wraps, rendered task checkboxes, sane heading scale, and a subtle selection style.
- [ ] **R9** Keyboard: j/k move, Enter opens, Esc closes, focus returns to the row.
- [ ] **R10** Tests assert **visible output** (text, and a CSS rule exists for every
      class rendered), not only class names (the backlog-02 audit lesson).
- [ ] **R11** (operator, 2026-09-27, after reviewing PR #135): **one unified
      header**. The Board and Backlog tabs always sit in the same place, on
      every screen (the run screen included, as plain links). Switching tabs
      happens client-side: the header element persists, and only its context
      area and the screen below it change. Each route keeps its own query
      (Backlog filters, Board `days`), Back/Forward switch tabs, and exactly
      one data stream is live (the active route's). `/` and `/backlog` serve
      one stylesheet, with backlog rules scoped so they never restyle the board.

### Refinement notes (variant A specifics)
- **Layout A:** state-major sections (blocked, waiting, ready, running,
  in-flight, other, closed) in the board's `.grp` language with a count badge;
  a sticky column header; one grid row per ticket; the row panel slides in
  from the right. Drift rows stay in their own state section, flagged `⚠`.
- **R6 data:** `RunView.reason` (the verbatim `run-end.reason`) is joined onto
  the row by `lastRunId` in `loadBacklog`. If the run was pruned or never
  ended, the row shows an honest "no run-end reason recorded".
  `waitsOn` is `deps.filter(!met)`, taken in `backlog-groups.ts`.
- **R7 data:** `drift` is derived in `loadBacklog`. It is time-free, so an
  idle backlog still pushes 0 bytes (v1.14 §8.4).
  `BacklogProject.provider` feeds the dispatch line: `just run-codex` for
  codex targets.
- **URL state** merges into the existing query and never drops `token`.
  Keyboard handlers are mounted only by `<Backlog/>` (the bundle is shared
  across screens) and ignore meta, ctrl and alt.
- **Markdown (R8)** fixes go in the shared `mdToHtml`. Every new class also
  gets a `runCss()` rule.
- **Spec:** v1.14 gets a change-log amendment covering the new §4 fields,
  D1 "by default" (filters are opt-in), and §8.1 (the store-less line moves
  behind the R3 toggle; the projection still yields one per target).

## Verify
- `bun run lint && bunx tsc --noEmit && bun run test`.
- Operator check in `adw web` `/backlog`: side-by-side with the approved prototype.

## Out of scope
- Mutating actions (still read-only, v1.14 R8). Goals and epic roll-up (supervisor Tier 4).
