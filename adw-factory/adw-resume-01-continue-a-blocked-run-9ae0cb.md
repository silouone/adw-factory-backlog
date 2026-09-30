---
id: adw-resume-01-continue-a-blocked-run-9ae0cb
type: feat
status: in-progress
priority: 2
created: 2026-09-28
depends: [adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69]
attempts: [{"runId":"adw-resume-01-continue-a-blocked-run-9ae0cb-1790759868841","branch":"adw/adw-resume-01-continue-a-blocked-run-9ae0cb","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-resume-01-continue-a-blocked-run-9ae0cb-1790759868841/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/165","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A blocked run can be continued from the node that failed, instead of thrown away

> **Refine at pickup.** This ticket needs a spec amendment before any test is
> written. Today's spec treats `blocked` as a terminal outcome (S6.1: "run to
> a terminal outcome"), and dispatch refuses any ticket whose status isn't
> `queued` (E7). Proposal first, operator approval, then Art. I.

## Why

2026-09-28, 18:36–18:49: a network outage blocked 10 runs (evidence table in
`adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69`). The factory's only
way forward was a fresh run from `dispatch`, which throws away everything in
the run's workspace:

| run | died at | left on disk |
|---|---|---|
| `cqc-fe-07-…-1790611419784` | build | +1619 / −163, 10 files |
| `cqc-fe-08-…-1790611451518` | build | +1286 / −298, 7 files |
| `cqc-be-03-…-1790611581162` | test | +963, 8 files |
| `adw-board-05-…-1790610062532` | review-spec | +433 / −131, gates passed |
| `cqc-be-02-…-1790611568176` | review-fix | +386, gates passed |
| `adw-card-02-…-1790609830123` | review-standards | +219, gates passed |

All six were salvaged by hand: one session, six parallel agents, each
re-deriving the lane's position from `journal.jsonl` and `artifacts/`. That
is the job this command does.

Even with adw-bug-30 in place, a build or test node that dies **dirty**
(fe-07, fe-08, be-03) is still refused a retry by `retryTransientIfClean`, and
correctly so, since a clean restart would double-apply edits. Only continuing
in place, in the same workspace, saves that work.

## Sketch (to be replaced by the refined requirements)

`adw resume <runId>` (and `just resume <runId>`):

1. Read the run's journal and find the last `node-end` with `outcome:
   "fail"` or the `run-end blocked`. Refuse loudly if the run ended
   `green`/`in-review`, or if its workspace or branch is gone.
2. Re-enter the lane **at that node**, in the same workspace, with a new
   runId linked to the old one (`resumedFrom`). Every earlier node's artifact
   stays in `ctx.data`, just as if those nodes had run.
3. An agent node seeded mid-way uses the existing hop machinery: checkpoint,
   then artifact, then synthesized (adw-bug-26/27). The resume source is
   journaled.
4. The ticket transitions `blocked → in-progress` in the ticket store, in one
   commit, and the attempt record gains `resumedFrom`.
5. Deterministic nodes (gates, commit, push, open-pr) are just re-run. They
   are idempotent, or can be made so.

## Open questions for refinement

- Is resume a new run with a link, or the same runId continued? This affects
  journals, the board and the §6 metrics.
- Which node types are safe to re-enter mid-way, and which must restart from
  their start?
- Does resume re-check that the base hasn't moved (the baseline green check),
  and rebase if it has?
- Could the operator give "resume from node X", for example to re-run a
  review after a hand edit?

## Verify (after refinement)

- The spec amendment is merged and cited.
- A red test drives a lane to `blocked` at `build` with a dirty workspace,
  resumes, and reaches `open-pr` without re-running `plan`, with the
  workspace diff preserved.
- `bun run lint && bunx tsc --noEmit && bun run test`, green.
