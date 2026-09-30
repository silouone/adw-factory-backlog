---
id: cqc-fe-17-four-things-that-render-broken-b6e409
type: bug
status: in-review
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-17-four-things-that-render-broken-b6e409-1790757351241","branch":"adw/cqc-fe-17-four-things-that-render-broken-b6e409","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-17-four-things-that-render-broken-b6e409-1790757351241/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/150","provider":"codex","model":"gpt-6-sol","rebased":"453a653c040d70f363e1e4290299ce74d9961c8c"}]
---
# Four parts of the Content quality page render broken, and three share one cause

Found by running the page against the dev API on 2026-09-30. Full audit:
adw-factory `ai_docs/2026-09-30-cqc-frontend-ux-audit.md`.

Three of the four come from **Go1d `View` being a flex column with `align-items: stretch`**,
which `foundations.md` states plainly and which this code does not counter.

## Base

**Base branch: `cqc/release-1`**, not `master`. Branch from it, target it.

Read first: `.agents/design-system/foundations.md` (Layout primitives),
`.agents/design-system/patterns.md` (Status vocabulary), and
`.agents/overlay/TESTING-NOTES.md` (passive effects do not flush after RTL `render`).

## 1. The drawer's tab bar stacks vertically

`RunDrawer/RunDrawerTabs.tsx:130` — the `View role="tablist"` has no `flexDirection="row"`,
so Checks / Evidence / Technical render as three full-width blocks and the selected tab
reads as a grey banner.

## 2. The check chips overlap the row below them

`RunDrawer/CheckStrip.tsx:36-60` — a two-line stacked `View` is placed inside
`ButtonMinimal size="sm"`. Go1d `Button` sets a **fixed 32px height** with
`alignItems: "center"` (`@go1d/go1d@0.9.95/build/components/Button/index.js:65,79`), so the
second line escapes the button box and collides with the chip beneath.

A button is not a layout container. Rebuild the chip so the control's height follows its
content — `patterns.md` renders status as a `Pill` with text; a chip that is both a status
and a control should be composed, not stuffed into a fixed-height button. Keep every
existing ARIA attribute (`aria-current="step"`, `aria-pressed`, `aria-controls`,
`aria-label`) and keep it keyboard-operable.

## 3. Three buttons stretch edge-to-edge and read as centred links

- `RunDrawer/RunDrawerHeader.tsx:222` "Open in player"
- `RunDrawer/RunDrawerHeader.tsx:289` "More" (checker summary)
- the drawer tabs, once (1) is fixed

Each sits directly in a column `View`, so it fills the width and centres its own label. The
user cannot tell these are controls.

## 4. The list page is ~1848px wide and overflows every normal screen

`CheckedContentList/columns.tsx` fixes seven column widths summing to **1720px**
(240+200+240+260+260+300+220), plus `PageBody`'s `paddingX={8}` = 128px.

**This ticket does not redesign the table** — `cqc-fe-18` does. Do the minimum that stops
the page overflowing a 1440px laptop: the `Case` (300px) and `Actions` (220px) columns hold
"Resolve…" and "Re-run" and are 520px between them. Bring the total under **1300px** without
removing a column or changing what a cell shows.

## Acceptance criteria

- [ ] The drawer tabs render as a horizontal tab bar; `role`, `aria-selected`,
      `aria-controls`, roving `tabIndex` and arrow-key navigation all still work.
- [ ] No check chip overlaps another at any drawer width down to 320 CSS px, with the
      longest label in `JOURNEY_ORDER` and a two-word status.
- [ ] "Open in player", "More" and each tab are sized to their label, not the container.
- [ ] The column widths total **≤1300px**; a test asserts the sum so it cannot silently
      regress.
- [ ] No new hex, `rgb(`, px spacing or px font size — this feature currently has none and
      must still have none (`foundations.md`).
- [ ] tslint, the full jest suite and `npm run build` green; `node scripts/start.js`
      compiles (the dev server's stricter `tsconfig.json`, see `cqc-fe-13`).

## Blocked by

- (nothing)
