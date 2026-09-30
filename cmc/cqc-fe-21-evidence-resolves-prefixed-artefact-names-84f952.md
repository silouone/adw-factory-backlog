---
id: cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952
type: bug
status: blocked
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952-1790794307578","branch":"adw/cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952-1790794307578/workspace","outcome":"blocked","provider":"codex","model":"gpt-6-sol"}]
---
# The evidence viewer finds artefacts on a real run: names arrive as `runs/<file>` and `output/<file>`

Found 2026-09-30 by reading both sides.

## The bug

- The backend lists every artefact with its layer prefix: `name: \`${layer}/${file}\`` for
  layers `runs` and `output` (CQC `backend/src/common/artefacts.ts:23-40`,
  `openapi.yaml:361, 987`).
- The CMC builds its inventory from `artefact.name` (`domain.ts:468`) and matches it against
  bare cited names:
  - `findInventoryName` uses an exact `indexOf` (`domain.ts:1365-1375`);
  - the snapshot regex is anchored at `^snapshots\/` (:1304);
  - the screenshot fallbacks are `final.png` and `agent-nav-go1-final.png` (:1355-1360).
- On a real run nothing resolves, and every citation reads "not archived".
- The tests hid this by using bare names (`RunDrawer/index.test.tsx:1939, 2022, 2100`).

## Red first (Art. I)

A domain test with a real-shaped inventory (`runs/agent-nav-go1-result.json`,
`runs/snapshots/page-0001.yml`, `output/playability-report.json`, …) and a report citing
bare names. Every citation must resolve. It is red today.

## Acceptance criteria

- [ ] Matching strips the `runs/` or `output/` layer before comparing. The resolved file keeps
      the full name, because that is what `GET …/artefacts/{name}` needs.
- [ ] When both layers hold the same file, follow the existing trust order. If that is
      undefined, prefer `output/`, the published layer. A test pins this.
- [ ] Snapshots and fallbacks work with prefixes.
- [ ] Existing tests are converted to prefixed names. No test keeps a shape the backend never
      sends.

## Blocked by

- (nothing)
