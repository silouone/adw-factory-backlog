---
id: adw-m2-02-open-pr
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m2
depends: [adw-m2-01-push-node]
attempts: []
---
# Open-PR node + PR body template

## Context

The PR is the factory's product and its report card: the description quotes
the run's headline stats straight from the journal (S1.5, S5.4). `gh` is
called directly (Art. VIII); the body template is a versioned prompt file
(N5).

## Deliverables

- `test/pipeline/nodes/open-pr.test.ts` (first, red — fake `gh` runner)
- `src/pipeline/nodes/open-pr.ts` · `prompts/pr-body.md`

## Requirements

- [x] Body rendered from `prompts/pr-body.md` + `readJournal(runId)`:
      ticket id + link, agent-written one-paragraph summary (from build
      node's retained final message), gate table (name/pass/duration),
      repair rounds used, total duration, run id, cLens session ids slot
      (empty until M3) (S1.5, S5.4, plan §5) — `renderPrBody` + template;
      per-gate duration is not derivable (`GateResult` = {name,outcome}, one
      gate node), so the suite delta renders instead (`{{gatesDuration}}`)
- [x] `gh pr create` as **ready-for-review** (not draft), base =
      `target.base`, head = attempt branch (S1.5) — node behaviour proven
      against a fake `gh`; end-to-end firing awaits lane wiring (see below)
- [x] On success: append attempt `{branch, pr, runId, outcome}` to the
      ticket's `attempts:` and transition status → `in-review`, each as its
      own commit (S1.6, N3, plan §5); the runId lets later invocations
      rehydrate workspace + session for the CI round (adw-m2-04) — attempt
      append FIRST (crash-safe), against a temp git fixture
- [x] PR-open failure (faked network/auth): bounded retries (3), then
      `fail(blocked)` with the branch intact and named in the journal (E6)
- [x] Rendering is a pure function (journal events → body string),
      snapshot-tested (Art. IX) — `computeRunStats` + `renderPrBody`

**Pending integration (team-lead's wiring pass, not this ticket's files):**
the open-pr node must be appended to `src/pipeline/lanes/chore.ts` after push
and its `OpenPrDeps` threaded from `src/cli.ts` (gh runner bound to
`cwd=repoPath`, `readJournal` bound to the runs root, the pr-body template
loaded, repoPath + defaultBranch). Until then the node is green in isolation
but not fired by a real `adw run`. Deps to thread are spelled out in the
builder report.

## Build protocol (Art. I)

1. Journal fixture → body snapshot; fake `gh` asserting args; failure/retry
   paths; ticket file mutations against a temp git fixture.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

CI polling (adw-m2-04), PR state sync (adw-m2-03).
