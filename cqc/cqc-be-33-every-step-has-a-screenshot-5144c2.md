---
id: cqc-be-33-every-step-has-a-screenshot-5144c2
type: feat
status: done
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-be-32-the-final-screenshot-and-scorm-log-are-published-dc3ce9]
attempts: [{"runId":"cqc-be-33-every-step-has-a-screenshot-5144c2-1790802711022","branch":"adw/cqc-be-33-every-step-has-a-screenshot-5144c2","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-33-every-step-has-a-screenshot-5144c2-1790802711022/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/47","provider":"codex","model":"gpt-6-sol","rebased":"1143fc0a7060f768e4834a226782a3ef39c4b4d4"}]
---
# Every navigation step has a screenshot next to its accessibility snapshot

Sources: FE D-8; CMC `spec-cqc-fe-release-1.md:334-336` (per-step screenshots were out of
scope for release 1), `backend-contract.md:138`.

## Today

- Each step has only an accessibility YAML: the agent calls `playwright-cli snapshot`
  (`scripts/e2b/nav-evidence.mjs:565, 606`).
- `copyRunSnapshots` keeps only `page-*.yml` (`scripts/runner/run-agent-nav-go1.mjs:87-96`).
- `verifyRepairRawSnapshots` counts only `.yml` (`publish-context-packs.mjs:118-131`).

## Change

- Take `snapshots/step-NNNN.png` with the same NNNN as the YAML.
- Hook it into the PATH shim `agentPathShim` (`nav-evidence.mjs:303-325`), after
  click/press/fill (the default, 2026-09-30).
- Cap it at **60 screenshots per run** (defaulted; the operator may override). Log what was
  dropped; don't truncate silently.
- Publish `snapshots/*.png` through the binary-safe path from `cqc-be-32`.

## Red first (Art. I)

- `scripts/runner/run-agent-nav-go1.test.mjs`: screenshots are copied with their snapshots.
- A `nav-evidence` selftest (the :827 pattern): the shim takes a screenshot after a click.

## Acceptance criteria

- [ ] The step PNG and YAML share a step number.
- [ ] The cap is enforced and logged.
- [ ] The repair and verify paths count both files.

**Runner image:** this changes code that runs in the container. Add a `COPY` line in `scripts/runner/Dockerfile:28-34` for every new module (it copies files one by one). After merge the operator rebuilds and pushes the image, then deploys with `--param runnerImageTag=<commit>` (`scripts/runner/README.md:19-26`). Say so in the PR body.

## Blocked by

- cqc-be-32-the-final-screenshot-and-scorm-log-are-published-dc3ce9
