---
id: cqc-fe-19-the-drawer-says-each-check-once-7a4d02
type: feat
status: blocked
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-17-four-things-that-render-broken-b6e409]
attempts: [{"runId":"cqc-fe-19-the-drawer-says-each-check-once-7a4d02-1790782788355","branch":"adw/cqc-fe-19-the-drawer-says-each-check-once-7a4d02","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-19-the-drawer-says-each-check-once-7a4d02-1790782788355/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/152","provider":"codex","model":"gpt-6-sol"}]
---
# The run drawer states each check once, and its facts in one place

Audit: adw-factory `ai_docs/2026-09-30-cqc-frontend-ux-audit.md`, Part 2.

## The problem

The drawer states the same six checks **three times**:

1. the check strip at the top (six chips with status);
2. "What this run could not observe" (bullets restating the unverified ones);
3. the per-check sections below (each check again with its status pill).

And it splits one fact block in two, with an action and a paragraph between the halves:

```
Checked 24 Sep 2026 · Revision 3 · Preview (tracking off) · Local
          [Open in player]
          Checker summary … More
Case: N/A · Days open: N/A · First detected: 24 Sep 2026 · Runs: 1 · Partner: N/A
```

The result is a drawer where nothing is ranked, and the reader scrolls past the same six
labels three times before reaching evidence.

## The moves

### 1. One fact block

Merge the two meta lines into a single group directly under the verdict, using the
information-display recipe (`.agents/conventions/references/recipes/information-display.md`):
label `fontSize={1} color="subtle"`, value in the default style. "Open in player" becomes an
action beside the verdict, not a divider between two halves of the same block.

### 2. Fold "what this run could not observe" into the checks it describes

Each limitation belongs to a check. Attach it to that check's section, so a reader who wants
to know why `Media plays` is `not verified` finds the reason in the `Media plays` section
rather than in a list six screens up. Keep the whole text — it is real data from the report
(`reasons`/`limitations`), not prose we own.

Where a limitation belongs to the run rather than a check (for example
`package_extraction_snapshot is empty`), keep it — once — above the check sections.

### 3. The strip navigates; the sections explain

After `cqc-fe-17` the strip is a working chip row. Let it be the only place the six statuses
are listed. The sections keep their own status pill — that is the anchor you land on — but
the third restatement goes.

## Acceptance criteria

- [ ] Each check's status appears in the strip and in its own section, and nowhere else.
- [ ] Every limitation string still reaches the user, attached to the check it explains, or
      to the run when it has no check. Nothing from the report is dropped — a test asserts
      each limitation from a fixture report is rendered exactly once.
- [ ] One fact block under the verdict; no meta line below the checker summary.
- [ ] Scrolling from the top of the drawer to the first piece of evidence passes each check
      label at most twice (strip, then section).
- [ ] The existing keyboard contract is intact: `j`/`k` run navigation, arrow-key tabs,
      focus moves to the check section when a chip is activated.
- [ ] The report-less run still renders (`NO_REPORT_MESSAGE`), and the imported-run banner
      still shows.
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Explicitly not in scope

- The Evidence and Technical tabs' internals.
- The list (`cqc-fe-18`).

## Blocked by

- cqc-fe-17-four-things-that-render-broken-b6e409
