---
id: cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4
type: feat
status: in-progress
priority: 2
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-fe-31-the-row-is-the-click-target-9222d4]
attempts: [{"runId":"cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4-1790879743229","branch":"adw/cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4-1790879743229/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"},{"runId":"cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4-1790883438641","branch":"adw/cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-32-the-top-of-the-page-is-one-calm-band-89bfe4-1790883438641/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/163","provider":"codex","model":"gpt-6-sol"}]
---
# The top of the page reads like the prototype: paste bar, summary with captions, one filter row, honest empty states

Design pass 2026-10-01. Evidence (operator-only, **not in your worktree** — do not look for it): adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`.
Today: `shots/appcfg-list-w1440-full.png`, `-filter0-open.png`, `-trigger-invalid.png`,
`-resolve-open.png` (the Resolved tab), and your own screenshots of the filter row and of
"Run checks". Target: `shots/proto-01-fullpage.png`, `proto-list-w1440-top.png`.

## The problems (measured)

- **Filters wrap onto two rows at 1440 and 1680** (one row only from ~1920). Search is
  `width={280}`, each `MultiSelect` `width={200}` with `marginLeft={5}`
  (`CheckedContentFilters.tsx:96`, `:192-194`) — 1400px against ~1250 available. The wrapped
  row starts 24px in (the per-item left margin). Search has no visible label, so its top aligns
  with the selects' **labels**, not their inputs. "Partner data is not available yet" is a
  separate line after the whole row (`:152-161`), under *Connected*, not under *Partner*.
- **Every Status option reads "(N/A)"** — "Fail-block (N/A)", "Needs review (N/A)", … The
  frontend reads `facets.status[value]` as a count (`:244-264`). The API returns facets as
  **arrays of present values**, no counts — `backend/openapi.yaml` `facets.status: array of
  ContentStatus` (measured: `{"status":["Fail-block","Needs-review"], …}`). The OpenAPI file is
  the contract (`cqc-fe-14`); `docs/cqc/backend-contract.md` 2b is stale on this point.
- **The Resolved tab, empty, with no filter set, says "No results match these filters" and
  offers "Clear all filters"** — because `hasActiveFilters` counts the tab itself
  (`useCheckedContentListParams.ts:48-54`).
- **"Run checks"** sits under the pagination: a 212px textarea (`TriggerChecks/index.tsx:262-277`
  — `flexGrow/minWidth` land on the inner `textarea`, not Go1d's bordered wrapper), and the
  estimate renders as one run-on line: "1 learning object  About $0.83  About 10–45 minutes  Up
  to 5 checks in parallel" (`:323-328`).
- **Summary cards have no captions**; the prototype explains each number.

## The target — the prototype's top band (variant B, chosen 2026-09-26)

```
Content quality
Known broken or unverified content, from the Content Quality Checker.                  ← description line

[ Paste LO ids from Slack to check them now: 37713131, 38663467 …            ] [Check]  ← full-width paste bar
  1 learning object · about $0.83 · 10–45 minutes · up to 5 in parallel                ← estimate, only when ids are valid

┌ OPEN ────────┐ ┌ FAIL-BLOCK ──┐ ┌ REVISION CHANGED ┐ ┌ PARTNERS AFFECTED ┐
│ 3            │ │ 1            │ │ 0                │ │ N/A               │
│ latest run   │ │ confirmed    │ │ needs a re-check │ │ needs provider    │      ← captions
│ not Pass     │ │ broken       │ │                  │ │ on runs           │
└──────────────┘ └──────────────┘ └──────────────────┘ └───────────────────┘

All (3)   Open (3)   Resolved (0)                                                     ← count badges
[🔍 Search LO id or run id] [Status: Any ▾] [Runner: Any ▾] [Authoring tool: Any ▾] [Connected: Any ▾] [Partner: N/A ⓘ]
```

Go1d translation and rules:

- **Paste bar first.** The trigger becomes a single full-width row directly under the page
  description: `TextArea` (one line high, grows to ~4 lines as ids are pasted) filling the width,
  the `Check` button on its right. Validation stays on submit/blur (`cqc-fe-20`); the error
  stays `Text color="danger"` with `aria-describedby`. The estimate renders as one line joined
  with ` · `, only when there is at least one valid id.
  **This reverses `cqc-fe-20`'s move of "Run checks" to the bottom** — the operator chose
  variant B, whose paste bar is the top of the page.
- **Summary cards**: keep the four, add the caption line (`fontSize={1}`, `color="subtle"`);
  label as eyebrow (`fontSize={0}`, uppercase). Labels route through the verdict-vocabulary
  helper `cqc-fe-27` added.
- **Tabs**: the count as a badge next to the label.
- **Filter row**: one `View flexDirection="row" flexWrap="wrap" alignItems="flex-end"` with a
  **gap** (`css={{ gap }}` with a spacing token, or a negative-margin wrapper) instead of
  per-item `marginLeft`. Labels inline in the control ("Status: Any") as in the prototype, or
  every control including search gets a visible label — pick one, apply to all six. Search
  flexes (`flexGrow={1}`, min ~220px, max ~320px); selects ~150–170px. Partner stays disabled; its
  reason moves into a tooltip / `aria-describedby` on the Partner control, not a free line.
  **Fits one row at 1440 with the sidebar.**
- **Facet options**: no count suffix. Options come from the facet arrays (present values); a
  value absent from the facet still appears (selectable) but carries no "(N/A)".
- **Empty states**: per tab, ignoring the tab itself as a filter — Resolved: "No resolved
  cases yet." Open: "Nothing open. Every checked LO passes or is resolved." "Clear all filters"
  appears **only** when a real filter (status, runner, authoring tool, connected, search) is set.
  The CTA is a normal-width button, not a 380px bar.

## Acceptance criteria

- [ ] Red tests first:
  - with the API's facet shape (arrays), no option label contains "(N/A)";
  - the Resolved tab with zero rows and no filter shows the per-tab empty copy and no
    "Clear all filters";
  - the trigger textarea's bordered wrapper fills the row (the flex props reach the wrapper);
  - the estimate is one text node joined with " · ";
  - the trigger renders above the summary cards.
- [ ] The trigger's existing behaviour is intact (parse, dedupe, submit, polling, disabled when
      the config says so — see `cqc-fe-33` for the config-down state).
- [ ] No new hex, `rgb(`, px spacing or px font size.
- [ ] tslint, jest and build green; the dev server compiles.

## Verify (operator)

Sweep `pass3-list.mjs` (`FIXCFG=1`) and `pass5-wide.mjs`: one filter row at 1440 and 1680.
