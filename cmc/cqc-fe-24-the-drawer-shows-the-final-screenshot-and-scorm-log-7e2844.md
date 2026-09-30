---
id: cqc-fe-24-the-drawer-shows-the-final-screenshot-and-scorm-log-7e2844
type: feat
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952]
attempts: [{"runId":"cqc-fe-24-the-drawer-shows-the-final-screenshot-and-scorm-log-7e2844-1790804552818","branch":"adw/cqc-fe-24-the-drawer-shows-the-final-screenshot-and-scorm-log-7e2844","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-24-the-drawer-shows-the-final-screenshot-and-scorm-log-7e2844-1790804552818/workspace","outcome":"in-review","pr":"https://github.com/go1com/domain-content-content-management-console/pull/156","provider":"codex","model":"gpt-6-sol"}]
---
# The drawer shows the final screenshot and the SCORM log as evidence

Sources: FE D-8, D-31; `open-questions.md:108-116`. Backend: `cqc-be-32` publishes the
final PNG and `scorm-log.json` (a bare array) and cites them by their own uri.

The `cqc-be-*` blocker below lives in the CQC ticket store (`~/adw/backlog/cqc`), so it is **not** in `depends:`, because `just next` only resolves ids in the same store. Check that it is `done` (merged **and deployed to dev**) before dispatching.

## Today

- The evidence viewer maps `screenshot → image` and `scorm_trace → scorm_table`
  (`domain.ts:1391-1402`). The image is rendered by `EvidenceViewer.tsx:134-142`.
- `parseScormTrace.ts:57-71` accepts a bare array, `{log: [...]}` or `observed.scormLog`,
  but **not** `observed.scormTrace`, which is what Go1 container runs have inline.
  Runs published before `cqc-be-32` therefore never render a trace.

## Red first (Art. I)

- `parseScormTrace` accepts `observed.scormTrace` (field names
  `{method, key, value, result, error}`, `go1-observers.mjs:155-162`).
- A drawer test: a prefixed inventory containing `runs/agent-nav-go1-final.png` and
  `runs/scorm-log.json` renders the image and the table.

## Acceptance criteria

- [ ] Both render for new runs.
- [ ] Old runs render the trace through `observed.scormTrace`.
- [ ] Old runs without a PNG still show "not archived".

## Blocked by

- cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952
- cqc-be-32-the-final-screenshot-and-scorm-log-are-published-dc3ce9 (CQC store)
