---
id: adw-m2-06-integration
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m2
depends:
  - adw-m2-01-push-node
  - adw-m2-02-open-pr
  - adw-m2-03-sync-pr-state
  - adw-m2-04-ci-round
  - adw-m2-05-operator-surface
attempts: []
---
# M2 integration — scratch-repo lifecycle exit gate

## Context

The epic's exit gate (plan §8 M2): the full lifecycle observed on a real
(scratch) GitHub repository, both the merge path and the rejection path.
Requires operator participation (the human merge — Art. IV). Still
construction-phase validation runs: traces/capture land at M3, so
success-metric 3 does not count them (plan §1 as amended).

## Deliverables

- `scripts/m2-shakedown.ts` — provisions/uses a scratch GitHub repo (with a
  trivial CI workflow) + toy chore ticket, invokes the real CLI
- Runbook results appended to this ticket's body, including measured wall
  time (N2 baseline)

## Requirements

- [x] Merge path: `adw run` → real push, real ready-for-review PR whose body
      quotes journal stats → ticket `in-review`; operator merges; next
      `adw run` → ticket `done` (S1.5, S1.6, S3.2)
- [x] Rejection path: second toy chore → PR closed without merge by the
      operator; next run → `rejected` with reviewer comments in the body;
      operator edits back to `queued` and verifies normal re-selection
      (S3.3 as amended, plan §5)
- [x] CI round observed at least once (force a CI-only failure, e.g. a check
      stricter remotely than locally) (S2.4)
- [x] Ticket file's git history shows every transition as its own
      `adw: <id> → <status>` commit (N3)
- [x] Wall time recorded ticket→PR; compare against the 30-minute NFR (N2)

## Verify

Both lifecycles completed on the scratch repo; journals + ticket history
tell the full story without rerunning (success metric 5).

## Runbook results (2026-07-15, scratch repo silouone/adw-m2-scratch)

Driven by `scripts/m2-shakedown.ts` against a real private GitHub repo with
a marker-tripped CI workflow. All three tickets are genuine toy chores
(write a haiku / limerick / a CI-tripping marker file). Construction-phase
runs — traces/capture land at M3, so success-metric 3 does not count them.

**Merge path — m2s-001 → done.** `adw run --ticket m2s-001`: dispatch →
provision → assemble-prompt → build → gates → commit → push → open-pr,
ending green with real PR #1 (ready-for-review, body rendered from the run
journal). Ticket → `in-review`. Operator merged #1; next `adw run` sync
swept it → `done`. Wall time to PR: **7.9 min** (first attempt blocked at
open-pr on a stale `GITHUB_TOKEN`, re-queued by operator edit; the second
attempt cut `adw/m2s-001-2` — the remote-collision fix, live).

**Rejection path — m2s-002 → rejected → re-queued.** `adw run --ticket
m2s-002` → PR #2, `in-review` (5.5 min). Operator closed #2 with a review
comment; next sync → `rejected`, the comment appended under a dated
`## Review feedback` section in the ticket body. Operator edit back to
`queued` verified normal re-selection.

**CI round — m2s-003, one round then blocked.** `adw run --ticket m2s-003`
→ PR #3 (4.5 min); GitHub Actions failed on the forbidden marker file
(stricter remotely than locally). A `sync` invocation found the failing
checks → charged `ciRounds:1` (committed before the agent) → resumed the
build session with the failing log → re-pushed → `ci-repaired`, ticket
stayed `in-review`, fresh `m2s-003-ci-*` journal written. CI failed again
(the marker is the ticket's own deliverable); the next `sync` → `blocked`
(single round spent), same finalizer shape as local blocking.

**N3:** every transition is its own `adw: <id> → <status>` commit
(`git log -- tickets/` shows the full chain incl. the ci-round charge).
**N2:** every ticket→PR wall time (4–8 min) is well under the 30-minute
bound. **Art. IV:** the factory never merged/closed — the human PR actions
were the operator's (performed via `gh` as an explicit one-off proxy on the
scratch repo; factory code stays merge-incapable, negative-capability guard
intact).

**Live findings folded back into source (red test first, then fix):**
1. `.claude/` session droppings — headless agent sessions write
   `.claude/data/sessions/*.json` into the workspace; the bare `git add -A`
   swept them into all three PRs. The commit + ci-commit sweeps and both
   dirty-workspace checks now scope to `-- . ':(exclude).claude'`
   (commit 8c7c0a3).
2. Stale `GITHUB_TOKEN` — the operator shell exports an invalid token that
   401s gh, and Bun does not propagate `process.env` mutations to spawned
   children, so it cannot be sanitized in-process; the shakedown script
   fails fast demanding an `env -u GITHUB_TOKEN` launch. `git push` was
   unaffected (credential helper via keyring). Watch item for M6: the live
   `adw` entrypoint (`src/cli.ts main`) should carry the same guard before
   real cLens runs.

## Out of scope

Traces/capture assertions (M3), real cLens tickets (M6).
