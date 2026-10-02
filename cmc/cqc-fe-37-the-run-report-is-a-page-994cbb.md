---
id: cqc-fe-37-the-run-report-is-a-page-994cbb
type: feat
status: in-review
priority: 1
created: 2026-10-02
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-34-nothing-renders-at-zero-px-3a4007]
attempts: [{"runId":"cqc-fe-37-the-run-report-is-a-page-994cbb-1790975731594","branch":"adw/cqc-fe-37-the-run-report-is-a-page-994cbb","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-37-the-run-report-is-a-page-994cbb-1790975731594/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/172","provider":"codex","model":"gpt-6-sol"}]
---
# The run report is a page

Design review 2026-10-02: `docs/cqc/design-review-2026-10-02.md` (overlaid in your worktree; read the Diagnosis, Decisions D-R1/D-R2 and the items cited below). Paths and lines are as of `cqc/release-1` @ `dc74178`.
Decision **D-R1**, items 6–13, 32, 38, 55.

## The problem

The run report (header, check strip, Checks / Evidence / Technical / Runs tabs) lives only in a
fixed `Drawer size="md"` (900 px, `RunDrawer/index.tsx:268-273`). On a 2000 px screen that leaves
1100 px of dimmed list; every pass since fe-17 compacted content inside the box.

## The target

- **New route** `/quality/:loId` (register next to `/quality` in `src/app/index.tsx:262-272`,
  same props: `apiBaseUrl`, `jwt`, `history`). It renders the **full run report** at page width
  under the CMC sidebar: header, check strip, tabs, footer actions. A run may be selected with
  `?run=<runId>` if the drawer already supports run selection (keep the same parameter name the
  drawer uses).
- **Extract one report body** used by both containers. Do **not** rename or move the existing
  `RunDrawer/*` files (parallel tickets edit them); add a container (e.g.
  `ContentQuality/RunReportPage/`) and pass a `layout: "peek" | "page"` prop where the body differs.
- **The drawer becomes a peek** (`?lo=` still opens it, j/k still steps through the list): header,
  check strip, the failing check's reason, actions, and a prominent **"Open full report"** link to
  `/quality/:loId`. The Checks/Evidence/Technical tabs move to the page only. Pick the narrowest
  existing `Drawer` size that fits this content (`components/Drawer/index.tsx:13-23`), and say in
  the PR which size and why.
- The page has a back link "Content quality" that returns to `/quality` with the list's filters
  preserved (the list keeps its query in the URL: `useCheckedContentListParams.ts`).
- Page layout: below the header and strip, the active tab content uses the full width; the
  Evidence tab's list + viewer gets the width the drawer could not give it.

## Acceptance criteria

- [ ] Red tests first:
  - visiting `/quality/10622213` renders the report heading, the strip, and the three tabs;
  - the drawer opened with `?lo=` renders no tabs and has a link to `/quality/<lo>`;
  - the page's back link targets `/quality` with the list query preserved;
  - an unknown LO id on the page renders the existing not-found / error state, not a blank page.
- [ ] Existing drawer tests that assert tabs are moved to the page, not deleted.
- [ ] No new hex, `rgb(`, px spacing or px font size; Go1d components and props verified per `.agents/conventions/references/go1d-verification.md`.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

At 1440 and 2000 px: the page has no horizontal overflow; the peek drawer shows the verdict and
the failing reason without scrolling.
