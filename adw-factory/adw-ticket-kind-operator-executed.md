---
id: adw-ticket-kind-operator-executed
type: chore
status: done
priority: 2
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-ticket-kind-operator-executed-1789129207576","branch":"adw/adw-ticket-kind-operator-executed","workspace":"/Users/silouane/adw-factory/runs/adw-ticket-kind-operator-executed-1789129207576/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/5","provider":"claude","model":"sonnet","ciRounds":1},{"runId":"adw-ticket-kind-operator-executed-ci-1789132675224","branch":"adw/adw-ticket-kind-operator-executed","workspace":"/Users/silouane/adw-factory/runs/adw-ticket-kind-operator-executed-1789129207576/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The ledger cannot say "a human does this, not an agent"

## Context

On 2026-09-11 `adw-m5-07-remote-ci-round-live-bar` was dispatched to the chore
lane by mistake. It is a BILLED E2B verification bar: it is discharged by a
human launching a remote run and OBSERVING a repair round inside a reattached
sandbox. No local build agent can satisfy it. A worktree run spent an agent
turn on an unsatisfiable task and blocked at gates.

Nothing in the ticket contract prevented that. The body said **operator-gated
and BILLED** twice; `just next` still listed it as the top runnable ticket,
because `next` reports `type` + `status`, which is all the contract has.

The vocabulary carries `type` (which lane runs it) and `status` (where it is).
It has no way to say **who executes it**. Verification bars, spikes, decisions
and billed live runs are all real backlog items that are not agent work — today
they are indistinguishable from a chore.

## To weigh at pickup — do NOT assume a design

- (a) A `kind: operator | agent` frontmatter field, defaulting to `agent`. The
      CLI refuses to dispatch `operator` with a named reason; `just next` hides
      them. Explicit, one field, but grows the contract (plan §5 amendment).
- (b) A reserved `type: bar` (or `manual`) with no registered lane, so the
      existing "no registered lane" refusal does the work. No contract change —
      reuses the mechanism that already refuses `epic`.
- (c) Convention only: a `> DO NOT DISPATCH` banner, as applied to m5-07 today.
      Zero code, zero enforcement — a human reading the ticket is the guard,
      and the CLI still happily dispatches it.

(b) is the smallest thing that actually enforces, and mirrors how `epic` is
already handled. (c) is what is in place right now and is not sufficient.

## Requirements

- [ ] Red test: dispatching an operator-executed ticket is REFUSED pre-engine,
      with a reason naming why, and zero tokens spent.
- [ ] `just next` / `just tickets` must not list them as runnable.
- [ ] `adw-m5-07-remote-ci-round-live-bar` converted to whichever design wins,
      and its interim `DO NOT DISPATCH` banner removed.

## Verify

- The red test fails before, passes after.
- `bun run lint && bunx tsc --noEmit && bun run test`.
- `just next` does not offer m5-07.

## Out of scope

Deciding the m5-07 bar itself — that stays owed and billed either way.

## Result (2026-09-11) — MERGED as PR #5 (`6602353`)

Implemented design (b): a reserved `type: manual` with no registered lane,
reusing the mechanism that already refuses `epic` rather than growing the
`TargetConfig` contract. The refusal names operator-execution specifically, so
it reads as a deliberate category and not a structural typo. Verified live:

    $ bun src/cli.ts run --ticket adw-m5-07-remote-ci-round-live-bar
    ticket "adw-m5-07-remote-ci-round-live-bar" is malformed (S1.3):
      - type: type "manual" is operator-executed, not agent-executed — a human
        discharges it directly; there is no lane and the CLI refuses to
        dispatch it

`adw-m5-07` is converted, and its interim DO-NOT-DISPATCH banner — convention
with zero enforcement — is gone, because the CLI now enforces it. The
mis-dispatch that wasted an agent turn this morning is structurally impossible.

**This run also exercised the CI-round machinery for the first time.** After
opening the PR it went `in-review`, `sync-pr-state` polled `gh pr checks`,
found them RED, and charged a ci-round (`e4fe139`) — the adw-m2-04 / m4-04
resume-on-CI-red path, which had never run against this repo because the
self-target had no checks until PR #4. It blocked after the round; the red was
the pre-existing Linux exec hang, not this ticket's code. Rebasing onto
`0a9a083` turned it green and it merged.

Closed by hand: `attempts:` records the BLOCKED run, so `sync-pr-state` has no
green attempt to reconcile.
