---
id: cqc-fe-31-the-row-is-the-click-target-9222d4
type: feat
status: in-progress
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-27-go1d-containers-stop-breaking-the-layout-96b868]
attempts: [{"runId":"cqc-fe-31-the-row-is-the-click-target-9222d4-1790865326022","branch":"adw/cqc-fe-31-the-row-is-the-click-target-9222d4","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-31-the-row-is-the-click-target-9222d4-1790865326022/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/162","provider":"codex","model":"gpt-6-sol"}]
---
# The whole row opens the run, the page never runs past the screen, and the row reads like the prototype

Design pass 2026-10-01. Evidence (operator-only, **not in your worktree** — do not look for it): adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`.
Today: `shots/appcfg-list-w1440-full.png`, `-hover-x360.png`, `appcfg-list-w1280-top.png`.
Target: `shots/proto-list-w1440-top.png`, `proto-list-w1440-hover-x700.png`.

## The problem (measured)

- **Hover and click land on the title only.** `elementFromPoint` across row 1 at 1440:
  pointer on the title (x=360) and Re-run (x=1300), `auto` everywhere else. The title is a
  `ButtonMinimal size="sm"` (`CheckedContentList/columns.tsx:271-288`) that paints a grey box on
  hover, indented 16px relative to the "LO id · tool" line under it. In the prototype the whole
  row is the target (pointer in every cell, row tint on hover, selected row highlighted while
  its drawer is open).
- **A row without a Go1 title opens through a button labelled "N/A"**
  (`CheckedContentList/index.tsx:336`), and its third line is a bare "N/A"
  (`formatConnection`, `columns.tsx:419-421`).
- **The page runs past the screen.** `ContentQuality/index.tsx:16` overrides
  `PageBody css={{ width: "auto", minWidth: "100%" }}`, so the page sizes itself from the
  table's fixed column widths (Go1d `TD/TH` px widths are `flexShrink: 0`). At 1440 the body's
  right edge is 1446px and the table card has no right border; at 1280 the Actions column, the
  fourth summary card and the pagination are clipped. On the Resolved tab (no rows) the page
  fits — so **the layout jumps when switching tabs**. The table's own `overflowX: auto`
  wrapper (`CheckedContentTable.tsx:45`) never gets to scroll.
- **Actions are ragged.** Row 1 wraps "Re-run · Resolve…" then "•••" onto a second line; rows 2–3
  show "Re-run •••" on one line. The `•••` menu has a single item, "Open case details".
- **The verdict says "failed" / "not verified"** while the summary card and the Status filter
  say "Fail-block" / "Needs review" — a user filtering "Needs review" sees rows labelled
  "not verified".

## The target — the prototype's row

```
CONTENT / PARTNER            STATUS                 AUTHORING TOOL  CONNECTED          FIRST DETECTED · LAST RUN  DAYS OPEN  RUNS  ENV      ACTIONS
LO 10622213                  [Fail-block]           [Unknown]       Static             23 Sep 2026                7 days     1     [local]  [Re-run] [Resolve…]
R U OK? (Hospitality)        ✓ ? ✕ ? ? ?                                               last 23 Sep 2026
Allara Learning · rev N/A
```

- **Primary**: `LO <id>` in bold, link colour on row hover. Under it, subtle `fontSize={1}`: the
  Go1 title (omitted when absent — never "N/A" as a title), then partner · revision.
- **Status**: the verdict pill (verdict vocabulary — the helper `cqc-fe-27` added; do not add
  another), the six-icon mini strip under it, an
  in-progress or failed-to-run line under that when relevant.
- **Authoring tool** and **Connected**: badges, with "+ Storyline embedded" / "served by <host>"
  as a subtle second line (data already on `ContentRow`; the filters already use it).
- **First detected · last run**: two lines. **Days open**: "7 days" for open cases, a "resolved"
  badge for resolved, `—` otherwise. **Runs**, **Env** (environment badge).
- **Actions**: `Re-run` and `Resolve…`/`Validate`/`Reopen` as small bordered buttons on one line
  (`white-space: nowrap`). Drop the one-item `•••` menu from the row; "Open case details" is
  what opening the row already does.
- **Whole row**: clicking anywhere in the row except an action opens the drawer; pointer cursor
  and a `faint` background on hover; the row whose drawer is open gets `soft` background and an
  `accent` left border, with `aria-current`. The keyboard target stays a real button (the LO id),
  so Tab + Enter still works and focus returns to it on close.
- **Width**: remove the `PageBody` width override. **Hard rule: the page never exceeds the
  viewport; the table scrolls horizontally inside its card.** Goal, not a gate: at 1440 the
  usable width is ~1070px (1440 − 242 sidebar − 128 `PageBody` padding); keep columns tight
  (badges, two short lines) so most of the table is visible, but never ellipsise the LO id or
  the verdict to make it fit — the prototype also scrolls its table at 1440.

### This deliberately reverses part of `cqc-fe-18`

`cqc-fe-18` cut the table to five columns on the 2026-09-30 audit's advice. The operator has
since compared the build with the approved prototype and asked for the prototype. The
prototype's nine columns come back, with the width rule above. If a column has no backing data
on dev, it renders `N/A` per the placeholder policy — do not drop the column.

## Acceptance criteria

- [ ] Red tests first:
  - clicking a non-action cell of a row opens the drawer for that LO; clicking Re-run does not;
  - the row opener's accessible name is `LO <id>` when the title is missing (never "N/A");
  - the open row has `aria-current`;
  - the verdict pill reads `Fail-block` / `Needs review` / `Pass`;
  - the row renders no `•••` menu;
  - `PageBody` has no `width: auto` / `minWidth` override.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

Sweep `pass3-list.mjs` (`FIXCFG=1`): `pastRightEdge` empty at 1440 and 1280; `cursorOverRow`
reports `pointer` at every x; switching All → Resolved does not change the summary cards' width.
