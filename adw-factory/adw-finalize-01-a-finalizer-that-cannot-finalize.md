---
id: adw-finalize-01-a-finalizer-that-cannot-finalize
type: bug
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-finalize-01-a-finalizer-that-cannot-finalize-1789407245166","branch":"adw/adw-finalize-01-a-finalizer-that-cannot-finalize","workspace":"/Users/silouane/adw-factory/runs/adw-finalize-01-a-finalizer-that-cannot-finalize-1789407245166/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/42","provider":"claude","model":"sonnet"}]
---
# `finalizeBlocked` throws, and leaves the ticket it was finalizing stranded

> Minted 2026-09-14 after this happened twice in one afternoon.

## Evidence

Twice on 2026-09-14 a run finished all its work — gates green, commit landed,
push landed — and then:

```
✗ adw-fe-04-heartbeat blocked — node "open-pr": open-pr refused: primary
  working copy is on "adw/sysprompt-02-preset-default", not the default branch
  "main" (N6)

error: status finalizeBlocked for "adw-fe-04-heartbeat": refuses to commit on
  "adw/sysprompt-02-preset-default": the default branch "main" must be checked
  out (plan §5)
      at onWrongBranch (src/intake/attempts.ts:103:9)
```

Same for `adw-tools-01-declare-the-agent-toolset`.

**Both guards are correct.** `open-pr` must not open a PR whose status commits
cannot land, and `commitTicketFile` must not commit off the default branch. The
defect is what happens next: `finalizeBlocked` **throws**, so the ticket is left
`status: in-progress` with `attempts: []` — no record that a run happened at
all, no PR reference, and the work invisible to the ledger.

That is precisely the state `adw-fe-04-heartbeat` exists to make detectable —
and `adw-fe-04` was one of the two victims.

Both were recovered by hand: PR #24 and PR #26, with the attempt records
written manually.

## The distinction this ticket turns on

**Do not remove the throw.** `attempts.ts` is explicit, and right:

> Dispatch-time cleanliness (E1) is NOT durable across a long run… re-checked
> before mutating, so an operator edit racing the finalize is never swept into
> the factory commit or overwritten. **The lesser evil in that rare race is a
> loud refusal (ticket left in-progress, workspace kept)** — never [a silent
> overwrite].

For the **dirty-tree race** that reasoning holds completely: refusing is better
than clobbering an operator's edit.

The **wrong-branch** case is different. Nothing is racing. No edit is at risk.
The repo is simply on another branch, and will not be on the right one by
itself. Refusing there converts a recoverable situation into a stranded ticket,
and the operator cannot even see that a run occurred.

## Requirements

- [ ] Establish the failure shape first: enumerate every way `finalizeBlocked`
      and `finalizeRejected` can fail to record a terminal outcome, from the
      code, and say which are races and which are not.
- [ ] A terminal outcome must be **recorded somewhere durable** even when the
      ticket file cannot be written. The run directory already exists and is
      already written to; the journal already has `run-end`. Nothing is missing
      — it is only that the TICKET is the sole place the operator looks.
- [ ] Preserve the dirty-tree refusal exactly as it is. It is load-bearing.
- [ ] The operator must be able to see, without reading a journal by hand,
      that a run terminated and could not finalize. Silence is the bug.
- [ ] Consider whether `adw-sync-01`'s sweep can adopt a stranded ticket from
      its run directory — the two tickets meet here.

## Verify

- A run whose finalize refuses still leaves a durable, discoverable terminal
  record naming the run, the outcome and the reason.
- An operator edit racing the finalize is still refused, never overwritten
  (the existing regression bar must keep passing).
- `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Making `open-pr` succeed off the default branch — that guard is correct.
