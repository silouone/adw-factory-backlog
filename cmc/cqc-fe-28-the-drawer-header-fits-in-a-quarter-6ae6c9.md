---
id: cqc-fe-28-the-drawer-header-fits-in-a-quarter-6ae6c9
type: feat
status: queued
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-27-go1d-containers-stop-breaking-the-layout-96b868]
attempts: []
---
# The drawer header takes a quarter of the drawer, not half, and reads like the prototype

Design pass 2026-10-01. Evidence: adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`
(`shots/appcfg-w1440-d0-checks-s0.png` is today; `shots/proto-w1440-d0-checks-s0.png` is the
target).

## The problem (measured at 1440×900, drawer 900px wide)

The header (`RunDrawer/RunDrawerHeader.tsx`) is pinned above `Drawer.Content` and is
**~475px tall**. The scrolling body below it is **343px** — a third of the drawer — before the
footer. The approved prototype's header is ~215px and its body ~500px.

The header holds: breadcrumb; `h2` LO id; a title line; the verdict pill + "Limited evidence"
headline + its sentence + "Open in player"; then **three fact rows on three different grids**
(4 columns, then 2 items at columns 1 and 3, then 3 columns — `RunDrawerHeader.tsx:283-305`,
`RunFact flexBasis="135px" flexGrow={1}` at `:326-327`, so nothing lines up); then
"Checker summary" + paragraph + "More".

## The target — the prototype's header, line by line

The approved design is "drawer 1, Verdict + tabs" (operator, 2026-09-26). Its header is five
short lines, every one of them a single row:

```
Content quality › LO 10622213   1 of 3                         [↑] [↓]  ✕     ← crumb line, small subtle text
LO 10622213  R U OK? (Hospitality)                                            ← h2; title inline, smaller, subtle
[Fail-block] Limited evidence ⓘ                    [Run ▾]  [Open in player]   ← verdict row
Checked 23 Sep 2026 · revision N/A · Preview launch (tracking off) · Local runner   ← one context line
CHECKER SUMMARY Agent-driven navigation of LO … (clamped to 2 lines)  More      ← label inline, eyebrow style
Open · 7 days open · first detected 23 Sep 2026 · 1 run · partner Allara Learning ← one case line
```

Go1d translation:

- **Crumb line**: one `View flexDirection="row" alignItems="center"`, `fontSize={1}`,
  `color="subtle"`. "Content quality" is an `InlineAction` (from `cqc-fe-27`) at the **same**
  size as the rest; the position "1 of 3" sits beside it; prev/next/close on the right.
- **Title**: `h2` holds the LO id; the Go1 title follows **on the same line** at `fontSize={2}`,
  `color="subtle"`. When there is no title, show nothing (not "N/A").
- **Verdict row**: one row. The verdict pill (status vocabulary, see below) then the claim
  strength as plain subtle text "Limited evidence" with an info tooltip carrying today's
  explanatory sentence ("One or more criteria could not be observed…"). Spacer. The run
  picker (only when there is more than one run). **"Open in player" becomes a bordered
  secondary `Button size="sm"`** — today it is a `ButtonMinimal` that reads as plain text.
- **Context line**: the facts *Checked, Revision, Launch mode, Environment* joined with ` · `
  in one `Text fontSize={1}`. `N/A` values stay `N/A` (placeholder policy).
- **Checker summary**: the label "Checker summary" inline as an eyebrow (`fontSize={0}`,
  uppercase, `color="subtle"`), the text clamped to 2 lines, "More"/"Less" as an `InlineAction`.
- **Case line**: *Case, Days open, First detected, Runs, Partner* joined with ` · `, the case
  state in bold. Hide "days open" and "first detected" when there is no open case.

Every fact that is on screen today stays on screen; it changes form, not presence. The
Technical tab already lists the full set as a grid.

### Status vocabulary

The verdict pill uses the **verdict** words the summary cards and the Status filter use:
`Fail-block`, `Needs review`, `Pass` (route through one helper — `domain.ts:365-375` maps the
verdict to the **check** words "failed / not verified" today, `CheckedContentSummary.tsx:15`
and `CheckedContentFilters.tsx:285-288` hard-code "Fail-block"). Check-level pills keep
"passed / failed / not verified". One helper decides which vocabulary; `cqc-fe-31` reuses it
in the list.

## Also in this ticket

- **D25 — the closed drawer is still in the tab order.** `src/components/Drawer/index.tsx:176-206`
  moves it off-screen with a transform and `aria-hidden`; its buttons (`Close LO …`,
  previous/next) remain focusable at x≈1464–2276. Fix it **in `RunDrawer`** (do not change the
  shared CMC `Drawer`): render no interactive header/content while closed, or set `inert`.
- **"Close LO N/A"**: the close button's label falls back to `LO <id>`, never `N/A`
  (`RunDrawer/index.tsx:134-136`).

## Acceptance criteria

- [ ] Red tests first:
  - the header renders exactly one context line and one case line, and no `dl` grid;
  - every fact rendered today (Checked, Revision, Runs, Partner, Launch mode, Environment,
    Case, Days open, First detected, checker summary) is still present in the header text;
  - the verdict pill reads `Fail-block` for a Fail-block run; check pills still read `failed`;
  - "Open in player" is a `Button`, not a `ButtonMinimal`;
  - a closed drawer has no focusable descendant (`querySelectorAll` of focusables is empty or
    the subtree is `inert`).
- [ ] The keyboard contract is intact: `j`/`k`, Esc, focus to the heading on open and on LO
      change, focus back to the row on close.
- [ ] Report-less runs and the loading/error shells still render the header.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

Sweep `pass2-app-drawer.mjs` (`FIXCFG=1`): the drawer report's `scroll.ch` (scroll body
height) goes from **343 → ≥ 560** at 1440×900, and the header is ≤ 240px tall.
(`cqc-fe-29` then pins the strip and tabs above the body; the prototype lands at ~500.)

## Explicitly not in scope

The strip, tabs and check cards (`cqc-fe-29`). The run picker's behaviour.
