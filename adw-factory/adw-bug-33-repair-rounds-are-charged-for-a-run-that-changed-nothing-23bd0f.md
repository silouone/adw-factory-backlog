---
id: adw-bug-33-repair-rounds-are-charged-for-a-run-that-changed-nothing-23bd0f
type: bug
status: in-progress
priority: 2
created: 2026-09-30
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-bug-33-repair-rounds-are-charged-for-a-run-that-changed-nothing-23bd0f-1790802810612","branch":"adw/adw-bug-33-repair-rounds-are-charged-for-a-run-that-changed-nothing-23bd0f","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-33-repair-rounds-are-charged-for-a-run-that-changed-nothing-23bd0f-1790802810612/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/168","provider":"claude","model":"claude-sonnet-5-5","rebased":"b131d4f21f6e81fbaf5f68fd23aa05998d3f739b"}]
---
# Repair rounds are charged for a run that changed nothing, or whose agent said it must not start

Measured 2026-09-30 on cqc-fe-22 and cqc-fe-26 (target `cmc`). Both build agents wrote in
`artifacts/build.md` that they were **blocked before implementation**: the backend
prerequisite was not done, and they changed no files. The pipeline still ran `test`, then
`gates`, then three repair rounds against an untouched base, and ended `blocked` with
"node gates exhausted 3 repair rounds". That cost about $1.50–1.80 and 30 minutes per
ticket. The block reason also blamed the gates, when the real reason was a prerequisite the
agent had already named.

## Two independent guards

1. **A tree identical to base can't have introduced a failure.** When `git diff <base>`
   (tracked and untracked, excluding `.adw/`) is empty and a gate fails, the failure is
   base-red by construction. Don't route to repair. End the run `blocked` with a reason
   saying the base is red at `<sha>`, name the gate, and flag the cached baseline for that
   sha as suspect (see adw-bug-31). This complements that ticket: here the base is proven
   red for free, with no reliance on the baseline cache.
2. **An agent's "not started" is a terminal outcome.** A build or plan node whose agent
   reports it refused to implement (a structured signal, e.g. a `status: not-started` field
   the build template asks for, not prose matching) ends the run as `deferred`. Record the
   agent's stated reason and leave the ticket `queued`. Never gate or repair it.

## Red first (Art. I)

- A pure classifier `(diffIsEmpty, gateResult) → route`: empty diff plus a failing gate gives
  `base-red`, not `repair`.
- A build-node test: an agent result carrying the not-started signal routes to the terminal
  deferral and never reaches `gates`.

## Acceptance criteria

- [ ] Empty diff plus a failing gate never charges a repair round, and the reason names the
      base sha and gate.
- [ ] The build and plan templates ask for the structured not-started signal. A run that
      emits it ends deferred, the ticket stays `queued`, and the agent's reason is in the
      journal and on the card.
- [ ] Spec amendment first if `deferred` is a new run outcome (Amendment rule).
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
