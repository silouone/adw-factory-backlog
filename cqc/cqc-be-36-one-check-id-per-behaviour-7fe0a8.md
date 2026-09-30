---
id: cqc-be-36-one-check-id-per-behaviour-7fe0a8
type: feat
status: in-progress
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: []
---
# The local and Go1 contracts name each check the same way

Sources: BE-28 (`docs/backend/decisions-2026-09-26.md:64`), `docs/backend/todo.md:52`.

**Build on PR #30 (`clxp-945-reviewer-agent`) once it is merged.** Reuse its binary-safe staging and scanning (`publish-context-packs.mjs`), `media-inventory.mjs` and `poc0/reviewer/*`. Do not re-implement them. PR #30 renames `media-plays` to `media-reachable` and makes it required. This ticket
finishes that reconciliation; it doesn't redo it.

## Today

- `REQUIRED_PLAYABILITY_CHECKS` (`poc0/observability/run-record.mjs:39-45`) lists
  completion-recorded, no-force-complete, navigation-happened, media-reachable and
  resume-survives-relaunch.
- The Go1 mapper emits `launch-renders` (:130, required, not in the list) and `media-plays`
  (:192, not required).
- `bind-and-merge.mjs:1144` references `launch-renders`.

## Red first (Art. I)

A cross-contract test: every id in `REQUIRED_PLAYABILITY_CHECKS` is emitted by
`playabilityReportFromNavRun` with `required_for_verdict: true`, or appears in an explicit,
commented exception list. It is red today on `media-reachable`.

## Acceptance criteria

- [ ] One media id: `media-reachable`, per PR #30.
- [ ] `launch-renders` joins the required set, or is listed as an exception with the reason.
      Record the choice in BE-28.
- [ ] Any Needs-review → Fail-block shift is listed in the PR as a verdict change.
- [ ] Historical runs keep their ids; no migration. The CMC already labels both ids
      (`domain.ts:345-353`).

**Runner image:** this changes code that runs in the container. Add a `COPY` line in `scripts/runner/Dockerfile:28-34` for every new module (it copies files one by one). After merge the operator rebuilds and pushes the image, then deploys with `--param runnerImageTag=<commit>` (`scripts/runner/README.md:19-26`). Say so in the PR body.

## Blocked by

- PR #30 (`clxp-945-reviewer-agent`) merged (not a ticket)
