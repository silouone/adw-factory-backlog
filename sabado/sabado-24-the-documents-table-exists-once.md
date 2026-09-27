---
id: sabado-24-the-documents-table-exists-once
type: feat
status: done
priority: 3
created: 2026-09-19
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-24-the-documents-table-exists-once-1789905468600","branch":"adw/sabado-24-the-documents-table-exists-once","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-24-the-documents-table-exists-once-1789905468600/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1232"}]
---
# refactor(documents): one useDocumentSelection hook behind the three document surfaces

> **Audit:** **P1-22** (axis C) — Lane J. **Schedule in a quiet window:**
> `SharedDocsPage.tsx` and `RecordDocumentsTable.tsx` are the two hottest
> files in the front (38 and 25 commits in 30 days). Fire it when no
> feature PR is open on the three files, or it is a week of rebases.

## What happens today

`frontend/src/pages/SharedDocsPage.tsx` (1 711 lines) and
`frontend/src/pages/assets/RecordDocumentsTable.tsx` (1 632) share **123
identical code lines** (>25 chars, comments stripped, `comm -12` on sorted
unique lines) — the whole selection/group state cluster: `selected`,
`selectedGroups`, `openGroups`, `confirmIds`, `groupIds`, `leaveConfirm`,
`barMounted`, `barOpen`, `deleteGroupsAndDocs`, `groupMode`, the row
`aria-label`s. `frontend/src/components/VaultEncryptedDocsPanel.tsx` (982)
shares another 76 with SharedDocsPage. The same comment sits verbatim at
`SharedDocsPage.tsx:1669` and `RecordDocumentsTable.tsx:1594`.
`RecordDocumentsTable.tsx:1453` already records a fix that lagged: "the
library made this change first (#1030) and this table was left behind".
Every fix to selection, grouping or the bar is made twice or diverges.

## Requirements

- [ ] **R1** One hook, `frontend/src/components/documents/useDocumentSelection.ts`:
      selection sets, group mode, confirm/leave state, bar mount/open, the
      delete-groups-and-docs orchestration — pure state and handlers, no
      JSX, no fetch (callers pass their mutations in).
- [ ] **R2** The three surfaces consume it. Renderers stay where they are;
      the `aria-label`s the tests query are unchanged.
- [ ] **R3** The three existing test files pass **unchanged**
      (`SharedDocsPage.test.tsx:263-330` pins the selection behaviour;
      `RecordDocumentsTable.test.tsx`; `VaultEncryptedDocsPanel.test.tsx`).
      One new test file for the hook itself.
- [ ] **R4** No behaviour change. No new DS component. The duplicated
      comment exists once, in the hook.

## Files

`frontend/src/components/documents/useDocumentSelection.ts` (new) ·
`frontend/src/components/documents/useDocumentSelection.test.ts` (new) ·
`frontend/src/pages/SharedDocsPage.tsx` ·
`frontend/src/pages/assets/RecordDocumentsTable.tsx` ·
`frontend/src/components/VaultEncryptedDocsPanel.tsx`. Nothing else.

## Verify

- [ ] The three existing test files: green before, green after, `git diff
      --stat` on them empty.
- [ ] Identical-line count between the two pages, measured the audit's way
      (`comm -12` on sorted unique lines > 25 chars, comments stripped),
      drops from 123 to under 30; between SharedDocsPage and the vault
      panel from 76 to under 20. Print both in the PR body.
- [ ] `wc -l` on the two pages drops by at least the lines the hook
      absorbed (no copy left behind).
- [ ] `ds:check` green; no new counter value moves.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.

## Out of scope

The `jscpd` ratchet (`sabado-23`); a single `documents` table (§6 #8);
`CalendarPage`'s model hook (§6 #7); any visual change.
