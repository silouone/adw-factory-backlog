---
id: cqc-fe-27-go1d-containers-stop-breaking-the-layout-96b868
type: bug
status: in-review
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: [{"runId":"cqc-fe-27-go1d-containers-stop-breaking-the-layout-96b868-1790857465215","branch":"adw/cqc-fe-27-go1d-containers-stop-breaking-the-layout-96b868","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-27-go1d-containers-stop-breaking-the-layout-96b868-1790857465215/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/160","provider":"codex","model":"gpt-6-sol"}]
---
# Four Go1d defaults break the page everywhere: pills wrap, buttons stretch or overflow, lists lose bullets, focus is invisible

Design pass 2026-10-01. Evidence (operator-only, **not in your worktree** — do not look for it):
adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`. This ticket is the **foundation** of the
pass: every other `cqc-fe-28..33` ticket depends on it. Fix the causes once, here, and apply
them at every site listed. Do not restyle anything else.

## The four root causes (verified in `node_modules/@go1d/go1d@0.9.95`)

- **R1 — a Go1d `Button`/`ButtonMinimal` has a fixed height** (md 40px, sm 32px,
  `alignItems/justifyContent: center`, `paddingX` 4). Multi-line content inside it overflows
  above and below the box and lands on the next element.
- **R2 — `ButtonMinimal` in a column `View` stretches to full width** (`align-items: stretch`)
  and centres its label; its `:hover, :focus` paints a grey band across the whole row. This
  is a large part of "hover is not on the right element".
- **R3 — Go1d's global CSS sets `list-style: none` and `outline: 0` on everything**
  (`build/foundations/globalCSS.js:39,62,63`). Lists lose bullets; keyboard focus is invisible.
- **R4 — `Text` forces the sans family even with `element="pre"`.** JSON renders proportional.

## Defects and sites

| # | Symptom | Site |
|---|---|---|
| D1 | Every status pill puts its icon **above** the word (two-line pill): list verdict, drawer verdict, check strip, check headings | `common/CqcStatusPill.tsx:44-51` — `Icon` renders a block-level flex `View` inside `Pill`'s inline `Text`. Also the hand-rolled pill in `RunDrawer/CheckStrip.tsx:70-78` |
| D4 | Breadcrumb "Content quality" is smaller and indented 16px vs "› Run details" | `RunDrawer/RunDrawerHeader.tsx:106` |
| D5 | "More"/"Less" under the checker summary is indented 16px and floats | `RunDrawerHeader.tsx:366-376` |
| D11 | Full-width centred text buttons with full-width grey hover: "Shared by every run of this profile", "Show/Hide declared and observed values", "Show raw report JSON", "Show metadata JSON", the trigger banner's "Try again" (measured 1001px wide) | `RunDrawer/ChecksTab.tsx:84-92`, `:183-196`; `RunDrawer/TechnicalTab.tsx:115-123`; `TriggerChecks/index.tsx:243` |
| D12 | Limitations, reasons and media lists render as indented paragraphs with no bullets | `ChecksTab.tsx:74-80`, `:93-106`, `:294`, `:425`, `:438` |
| D13 | Declared/observed JSON and raw JSON render in the proportional font, no code block | `ChecksTab.tsx:483-497`; `TechnicalTab.tsx:125-139` |
| D14 | Programmatic focus on "What this run could not observe" draws a **full-width** blue box | `ChecksTab.tsx:63-69` (and the section at `:140`) — a `:focus` box-shadow on a block heading, fired by `RunDrawerTabs.tsx:89-96` |
| D24 | No visible focus ring on page tabs, search, filter selects (computed outline none, box-shadow none) | R3; Go1d `Tab`, `TextInput`, `MultiSelect` add none back |
| D3t | Technical tab: the last row of facts spreads to half-width ("Positions observed" at the midpoint, not under a column) | `TechnicalTab.tsx:83-84` — `flexBasis="220px" flexGrow={1}` makes each row share its own width |

## The fix, as primitives (in `src/components/ContentQuality/common/`)

1. **`CqcStatusPill`**: icon and label on one line — a `View flexDirection="row" alignItems="center"`
   inside the `Pill`, or the `Icon` made inline. Same for the strip's pill.
2. **An inline text action** (name it in the codebase's idiom, e.g. `InlineAction`): a
   `ButtonMinimal` with `paddingX={0}`, `css={{ alignSelf: "flex-start" }}`, auto height,
   hover/focus confined to its own label width. Disclosures ("Show …", "Shared by every run…",
   "More/Less") get a chevron `iconName` so they read as disclosures. Replace every D4/D5/D11 site.
3. **Bulleted list**: a `ul` with `css={{ listStyle: "disc", paddingLeft: … }}` and `li` items
   that render as list items (not flex columns). Replace every D12 site.
4. **Code block**: `Text element="pre" fontFamily="mono"` with `backgroundColor="faint"`,
   `padding={3}`, `borderRadius={2}`, horizontal scroll. Replace every D13 site.
5. **Focus**: one scoped rule on the Content quality root —
   `:focus-visible { box-shadow: 0 0 0 2px <accent token> }` — and **no** ring for
   `tabIndex={-1}` programmatic targets (`:focus:not(:focus-visible) { box-shadow: none }`).
   Remove the D14 `:focus` box-shadow.
6. **Verdict vocabulary (D23)**: one helper decides the words. The **verdict** of a run/LO
   reads `Fail-block`, `Needs review`, `Pass` — the words the summary cards and the Status
   filter already use. A **check** reads `passed`, `failed`, `not verified`. Today
   `domain.ts:365-375` maps the verdict to the check words, so a "Needs review" filter shows
   rows labelled "not verified"; `CheckedContentSummary.tsx:15` and
   `CheckedContentFilters.tsx:285-288` hard-code "Fail-block". Add the verdict-label helper
   next to `getStatusLabel` and route the summary card and filter labels through it.
   `cqc-fe-28` (drawer verdict) and `cqc-fe-31` (row verdict) only **consume** it.
7. **Fact grid**: a `FactGrid` that lays label/value pairs on **fixed columns**
   (`width={[1, 1/2, 1/3]}` or a CSS grid, `flexGrow={0}`), so a short last row stays under its
   columns. Apply it to `TechnicalTab`. (`cqc-fe-28` applies it to the drawer header.)

**A rule for the whole feature, write it as a comment on `InlineAction`:** a Go1d `Button`
or `ButtonMinimal` never wraps more than one line of content. A clickable block of several
lines is a `View element="button"` with auto height.

## Acceptance criteria

- [ ] Red tests first. jsdom has no layout, so assert structure and computed style **on
      surfaces that render today** (not on the new primitives' imports, which do not exist yet).
      Emotion styles are in the jsdom document: use the pattern already in
      `RunDrawer/index.test.tsx` — `window.getComputedStyle(el).<prop>` — not `toHaveStyleRule`
      (not configured here).
  - `CqcStatusPill` renders its icon and label as siblings inside one row container.
  - The "Shared by every run of this profile", "Show declared and observed values" and
    "More" buttons have `padding-left: 0` and `align-self: flex-start` (computed style).
  - The verdict helper returns `Needs review` for `Needs-review`; the summary card and the
    Status filter option render through it.
  - Limitations render as `li` inside a `ul` whose style has `list-style: disc`.
  - Declared/observed JSON renders in a `pre` with the mono family.
  - The tabpanel heading focused programmatically has no box-shadow outside `:focus-visible`.
- [ ] No new hex, `rgb(`, px spacing or px font size; tokens only.
- [ ] No change of copy, order or behaviour beyond the listed defects.
- [ ] tslint, jest and build green; the dev server compiles.

## Explicitly not in scope

Rendering the verdict words in the drawer and the row (`cqc-fe-28`, `cqc-fe-31`). Header layout (`cqc-fe-28`), strip/tabs/check cards (`cqc-fe-29`), evidence list
(`cqc-fe-30`), list row (`cqc-fe-31`), filters/trigger (`cqc-fe-32`), case actions (`cqc-fe-33`).
They all build on these primitives.

## Verify (operator)

Re-run the sweep (`ai_docs/2026-10-01-cqc-fe-design-pass/sweep/pass2-app-drawer.mjs`, with
`FIXCFG=1`): no two-line pill in any capture; no full-width grey band on hover of a disclosure.
