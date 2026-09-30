---
id: cqc-be-32-the-final-screenshot-and-scorm-log-are-published-dc3ce9
type: feat
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: []
---
# The final screenshot and the SCORM log are published, under the names the report cites

Sources: FE D-8, D-5, D-31; CMC `open-questions.md:108-116` (17a/17b), `backend-contract.md:128-139`
(its line refs are stale).

**Build on PR #30 (`clxp-945-reviewer-agent`) once it is merged.** Reuse its binary-safe staging and scanning (`publish-context-packs.mjs`), `media-inventory.mjs` and `poc0/reviewer/*`. Do not re-implement them.

## What's broken

- `RAW_KEEP` (`scripts/context-extract/publish-context-packs.mjs:74-86`) keeps no `*.png`,
  no `scorm-log.json` and no `enrolment-t*-summary.json`.
- `stageRedacted` (:155-166) and `scanTree` (:46-60) read every file as utf8, so a PNG would
  be corrupted or flagged. PR #30 fixes this.
- The runner writes `agent-nav-go1-final.png` (`scripts/e2b/sandbox-agent-nav-go1.mjs:405`)
  and, in launch mode, `agent-nav-go1-relaunch.png` (:427). Neither is published.
- The local runner writes `final.png` (`scripts/local/run-local-nav.mjs:295`), but the mapper
  always cites `agent-nav-go1-final.png` (`poc0/observability/playability-from-nav-run.mjs:78`).
- No runner writes `scorm-log.json`. The trace sits inline as `observed.scormTrace`
  (`scripts/e2b/go1-observers.mjs:155-162`) and is cited as
  `agent-nav-go1-result.json#observed.scormTrace` (mapper :83).
- Cited but never published: `enrolment-t0/t2-summary.json` (:73-74) and
  `agent-stdout.txt` (:79).

## Red first (Art. I)

- Publisher selftest (fixtures ~:520-532):
  - a PNG survives staging byte for byte (equal sha) and is not flagged;
  - `rawFilesFor` includes the final PNG and `scorm-log.json`.
- Mapper test: every cited `uri` names a file the run actually wrote.

## Acceptance criteria

- [ ] The final PNG (and the relaunch PNG in launch mode) is published, byte-exact.
- [ ] `scorm-log.json` is written as a bare array with fields
      `{method, key, value, result?, error?}` (the CMC parser reads `m|method`, `k|key`,
      `v|value`), published, and cited by its own uri.
- [ ] Every cited file is either published or no longer cited.
- [ ] Local runs cite the screenshot name they actually wrote.
- [ ] PNGs can't be secret-scanned. The PR says so, and the operator accepts it before merge,
      because a PNG shows partner content.

**Runner image:** this changes code that runs in the container. Add a `COPY` line in `scripts/runner/Dockerfile:28-34` for every new module (it copies files one by one). After merge the operator rebuilds and pushes the image, then deploys with `--param runnerImageTag=<commit>` (`scripts/runner/README.md:19-26`). Say so in the PR body.

## Blocked by

- PR #30 (`clxp-945-reviewer-agent`) merged (not a ticket)
