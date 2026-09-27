---
id: adw-caps-03-a-mid-stream-breach-still-salvages
type: bug
status: done
priority: 2
created: 2026-09-27
caps: {minutes: 120, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-caps-03-a-mid-stream-breach-still-salvages-1790493133716","branch":"adw/adw-caps-03-a-mid-stream-breach-still-salvages","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-caps-03-a-mid-stream-breach-still-salvages-1790493133716/workspace","outcome":"blocked"},{"runId":"adw-caps-03-a-mid-stream-breach-still-salvages-1790516196379","branch":"adw/adw-caps-03-a-mid-stream-breach-still-salvages","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-caps-03-a-mid-stream-breach-still-salvages-1790516196379/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/128","provider":"claude","model":"sonnet"}]
---
# A mid-stream turn-ceiling breach blocks the run and discards finished work

> **Evidence, 2026-09-27.** After `adw-bug-23` (turns counted at all) and
> `adw-bug-25` (a turn is an API call), a single agent node's own usage can no
> longer breach the run ceiling at a node **boundary**. The mid-stream ceiling
> always fires first, and it **blocks** the run. So `adw-caps-01`'s
> deterministic tail (gates → commit → push after a breach) now applies only
> when turns are summed across several sessions (review-fix). Runs
> `adw-usage-04-…` and `adw-perf-06-…` were blocked mid-build this way, with
> uncommitted work left in the worktree.
> `adw-bug-25` (PR #119) re-scoped the ci-round test that assumed a boundary
> breach. Orchestrator decision 2026-09-27: accept "the tail covers boundary
> breaches only" for now, and file this ticket.

## Requirements

- [ ] **R1 — red first.** A lane test in which `build` breaches the run
      ceiling mid-stream with a dirty worktree: today the run ends `blocked`
      with nothing committed.
- [ ] **R2 — route the breach through the tail.** A mid-stream ceiling
      breach ends the node like a boundary breach does (caps-01): the
      deterministic tail runs (gates; commit and push, then open a **draft**
      PR if the gates are green), and the run outcome still records the
      breach.
      *Amended 2026-09-27 (orchestrator): "draft" was an authoring slip. Spec S1.5 (`adw-v1.md`) requires a **ready-for-review** PR, pinned by `open-pr.test.ts`; the tail opens a normal PR. The review-fix agent's won't-fix in #128 is accepted.*
- [ ] **R3 — no double spend.** No agent stage runs after the breach, only
      the deterministic nodes.
- [ ] **R4 — spec.** Amend `adw-caps-01`'s spec/ticket text to cover the
      mid-stream case, and cite this ticket.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`.
