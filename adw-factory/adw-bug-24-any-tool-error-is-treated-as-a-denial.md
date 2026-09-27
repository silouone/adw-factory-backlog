---
id: adw-bug-24-any-tool-error-is-treated-as-a-denial
type: bug
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# Any failed tool call is treated as a permission denial and kills the node

> **Evidence, 2026-09-27, run `adw-bug-22-a-poisoned-session-throws-away-ninety-minutes-of-work-1790470050008`.**
> The `plan` node sent a malformed `Read` call. The SDK answered with an ordinary
> recoverable `tool_result` `is_error: true`:
> `<tool_use_error>InputValidationError: Read was called with input that could not be parsed as JSON…`.
> The stream consumer journaled `tool-denied` and failed the node: **run blocked**.
> The model would have retried. For comparison, `adw-store-01-tickets-dir-1790448708493`'s
> transcript holds **22** `is_error` tool_results in one session, all
> ordinary (`Exit code 1`, `InputValidationError`, a Bash safety refusal).
> Under today's code, the first one of those kills the run.

> **Amendment, 2026-09-27 — evidence corrected, decision recorded.**
> The "all ordinary" claim above is wrong. Measured: **14 of adw-store-01's
> 22** `is_error` results are CLI-stamped `toolDenialKind: "user-rejected"`
> approval rejections ("This command requires approval", "…require(s)
> approval: …", the shell-syntax and cd-before-git refusals), and the agent
> **worked through them** (`plan` ended `next` after 9). The run survived
> only because its factory binary predated `adw-bug-20`. sabado-25's agent
> likewise worked through its 8 approval rejections and stopped only on
> "The user doesn't want to take this action right now. STOP…".
> **Decision (orchestrator, for the operator):** only that STOP text fails
> the node. Every approval rejection is an ordinary, retryable error, bounded
> by the no-progress detector. R2/R3 below are read under this amendment;
> `adw-bug-20`'s R1a test keeps its approval fixture, now asserting `next`,
> plus a STOP-text fixture that still fails.

## Root cause

`src/pipeline/nodes/build.ts` (`consumeAgentStream`, the `message.type === "user"`
branch added by `adw-bug-20` R1a) fails on **any** `tool_result` with
`is_error: true`, except suite-guard's exact reason string. `adw-bug-20` R1a
asked for an `is_error` tool_result **"carrying a `user-rejected` denial"**.
The qualifier was dropped.

## Requirements

- [x] **R1 — red first.** A `consumeAgentStream` test feeds an `is_error: true`
      tool_result whose content is an ordinary tool failure (`Exit code 1…`,
      and the `InputValidationError` text above, verbatim), followed by a
      normal result. The node must end `next`, not `fail`. This fails on `main`.
- [x] **R2 — only a real denial fails.** Identify a permission denial by its
      actual wire shape, measured from a real transcript. `sabado-25`'s build
      session (see `adw-bug-20`) has 8 `user-rejected` denials. Candidates are
      the SDK's own marker on the message or block (e.g. `toolUseResult` /
      `toolDenialKind`), or the fixed denial texts ("The user doesn't want to
      take this action right now", "…require approval…"). Prefer a structural
      marker over text if one exists on the **stream** message the SDK
      yields, not just in the on-disk transcript. State in a comment which one
      was used and the evidence.
- [x] **R3 — the bug-20 tests stay green**: a real denial still fails,
      naming the tool and rule; suite-guard's denial still does not.
- [x] **R4 — a run of the ordinary errors.** Five consecutive ordinary
      `is_error` results do not fail the node. The no-progress/loop detectors
      remain the only bound on repeated failure.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, then re-dispatch
`adw-bug-22` through the factory.

## Blocked by

None. Priority 0: every factory run is exposed.
