---
id: cqc-fe-29-the-strip-and-tabs-stay-put-and-checks-are-cards-31f799
type: feat
status: in-progress
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-28-the-drawer-header-fits-in-a-quarter-6ae6c9]
attempts: [{"runId":"cqc-fe-29-the-strip-and-tabs-stay-put-and-checks-are-cards-31f799-1790879734896","branch":"adw/cqc-fe-29-the-strip-and-tabs-stay-put-and-checks-are-cards-31f799","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-29-the-strip-and-tabs-stay-put-and-checks-are-cards-31f799-1790879734896/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# The check strip and the tabs stay put, and each check is a card you can read at a glance

Design pass 2026-10-01. Evidence (operator-only, **not in your worktree** — do not look for it): adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`.
Today: `shots/appcfg-w1440-d0-checks-s0.png`, `-s2.png`, `-chip2-click.png`. Target:
`shots/proto-w1440-d0-checks-s0.png`, `-s1.png`.

## The problem

- **The strip wraps.** Six chips sized by content wrap to two rows at 900px (the sixth alone
  on row two), unequal widths, and **hover looks identical to selected**: hover uses
  `colors.faint`, selected uses `soft` — both near-grey — and a `:focus` accent shadow stays on
  a clicked chip, so two chips look selected at once (`RunDrawer/CheckStrip.tsx:20-28`,
  `:39-66`).
- **The tabs are grey boxes** (`ButtonMinimal active`, `RunDrawerTabs.tsx:157-171`) while the
  page tabs are underlined Go1d `Tab`s. And the strip and tabs live **inside** the scrolling
  `Drawer.Content`, so they scroll away (`RunDrawer/index.tsx:287-296`).
- **Check sections read as disabled and break**: each check name is a large **grey** `h3`
  inside a `ButtonMinimal` (inherits `color="subtle"`); the focused check's grey bar is 40px tall
  while its content is not — the title pokes above it and the pill hangs below
  (`ChecksTab.tsx:142-164`). No section says *why* until you expand it.
- **Shared limitations are hidden** behind a centred "Shared by every run of this profile" button.

## The target — the prototype's drawer body

```
┌──────────────────────── pinned (does not scroll) ─────────────────────────┐
│ ✓ Launches  │ ? Learner can │ ✕ Completion │ ? No forced │ ? Media  │ ? Resume │  ← strip: 6 equal cells
│   and renders│   navigate    │   is recorded│  completion │   plays  │  survives│     icon left, label bold,
│             │   not verified│   failed     │ not verified│not verif.│  not ver.│     status word under it
│─────────────┴───────────────┴══════════════┴─────────────┴──────────┴──────────│  ← failed cell: danger underline
│ Checks  1 failed · 4 not verified │ Evidence  3 of 4 available │ Technical   │  ← underlined tabs + count badge
└───────────────────────────────────────────────────────────────────────────┘
  WHAT THIS RUN COULD NOT OBSERVE  (6)                                          ← eyebrow + count
  • Agent declared stop reason "END_OF_COURSE" …                               ← own limitations, bulleted
  ┌ Also on most runs of this profile: ───────────────────────────────────┐   ← shared, tinted box, VISIBLE
  │ • package_extraction_snapshot is empty …                              │
  └───────────────────────────────────────────────────────────────────────┘
  ┌───────────────────────────────────────────────────────────────────────┐
  │ ✓ Content launches   passed                                         › │   ← compact card: one line
  └───────────────────────────────────────────────────────────────────────┘
  ┌───────────────────────────────────────────────────────────────────────┐
  │ ? Navigation happens  No content-position transition observed; …    › │   ← first reason, one line, ellipsis
  └───────────────────────────────────────────────────────────────────────┘
  ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
  ┃ ✕ Completion is recorded  failed              Blocker · counts toward verdict ┃  ← focused card: expanded
  ┃ • no completion/passed write or derived SCORM 1.2 terminal status observed ┃
  ┃ ▸ Declared vs observed                                                ┃
  ┃ Evidence: [SCORM trace]                                               ┃  ← evidence as chips
  ┃ (inline viewer of the selected evidence)                              ┃
  ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
```

The drawing shows the prototype's wording; **check labels stay exactly as today** (they come
from the report: "Content launches", "Navigation happens", …). Do not rename checks.

Go1d translation:

- **Layout**: header (from `cqc-fe-28`), strip and tab bar are pinned siblings above the one
  scrolling element; only the tab panel scrolls. (Either move them out of `Drawer.Content` or
  make them `position: sticky; top: 0` inside it, with a background so text does not show through.)
- **Strip**: a `ul` of six cells, `flexWrap="nowrap"`, each cell `flexGrow={1} flexBasis={0}
  minWidth={0}`, a `View element="button"` (auto height — rule from `cqc-fe-27`), divider
  between cells, icon on the left spanning two lines, label `fontSize={1}` bold, status word
  `fontSize={0}` subtle (omit the word for `passed`). Failed cell: a `danger` bottom border.
  Selected cell: `soft` background + `accent` bottom border. Hover: a background distinct from
  selected. Focus ring only on `:focus-visible`. Long labels wrap inside the cell, never the strip.
- **Tabs**: Go1d `TabNavigation` + `Tab` with `isSelected` (the page's own tab style), keeping
  today's ARIA (`role="tablist"`, roving `tabIndex`, arrow keys). Each tab label carries a count
  badge: Checks → "1 failed · 4 not verified" (or "all passed"); Evidence → "3 of 4 available".
  Replace today's "Evidence (3 of 4 available)" parenthesis.
- **Limitations section**: eyebrow heading with a count badge; the run's own limitations as a
  bulleted list (`cqc-fe-27` primitive); shared limitations **shown by default** in a tinted box
  (`backgroundColor="faint"`, border, radius) under the caption "Also on most runs of this
  profile:". Keep the grouping logic from `cqc-fe-23`.
- **Check cards**: each check is a bordered card (`border={1} borderRadius={3}`), one line when
  collapsed — status icon, **name in the default text colour** (not subtle), status word, the
  first reason (single line, ellipsis), a chevron. Failed card: danger-tinted border and faint
  danger background. The focused card is expanded: reasons as bullets, severity meta right-aligned
  on the title row ("Blocker · counts toward verdict"), a "Declared vs observed" disclosure, the
  evidence as chips, the inline viewer. Clicking a strip cell focuses its card (today's
  behaviour); "Show all checks" returns to the list.

## Acceptance criteria

- [ ] Red tests first:
  - the strip renders 6 cells in one `nowrap` row container; the selected cell exposes
    `aria-current`; hover/selected use different tokens (assert the style rules);
  - the tablist is rendered with Go1d `Tab` and each tab label includes its count text;
  - strip and tablist are **not** descendants of the scrolling element (or are `sticky`);
  - each check card's collapsed row contains the first reason; the name is not `subtle`;
  - every limitation in a fixture report is rendered exactly once, shared ones visible without
    a click.
- [ ] Keyboard contract intact: arrow-key tabs, `j`/`k`, strip activation focuses the card.
- [ ] The report-less run still renders (`NO_REPORT_MESSAGE`); the imported-run banner still shows.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

Sweep `pass2-app-drawer.mjs` (`FIXCFG=1`): strip on one row at 900px; tabs visible in every
`-s1..s5` scroll capture; `scroll.ch` ≥ 470.

## Explicitly not in scope

The Evidence tab and the evidence list rows (`cqc-fe-30`). The Technical tab.
