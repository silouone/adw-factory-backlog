---
id: adw-bug-20-the-denial-fix-misses-and-a-ticket-can-never-finalize
type: bug
status: done
priority: 1
review: false
created: 2026-09-21
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-bug-20-the-denial-fix-misses-and-a-ticket-can-never-finalize-1790030703390","branch":"adw/adw-bug-20-the-denial-fix-misses-and-a-ticket-can-never-finalize","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-20-the-denial-fix-misses-and-a-ticket-can-never-finalize-1790030703390/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/108","provider":"claude","model":"sonnet"}]
---
# The denial fix watches the wrong message, and a ticket without `attempts:` can never be finalized

> Two independent defects surfaced by one run,
> `sabado-25-the-built-front-boots-in-a-browser-before-it-ships-1789996074209`.
> Both stop a run terminating cleanly. R1 supersedes the unshipped half of
> `adw-bug-19`.

---

## R1 — `adw-bug-19` shipped detection for a message shape that never arrives

`adw-bug-19` was filed with three requirements. **Only R2/R3 shipped (PR
#103). R1 — the prevention — did not.** `permissionMode` is still pinned
`"acceptEdits"` at `src/pipeline/nodes/build.ts:840` (and typed at `:135`).

Worse, the detection that *did* ship watches the wrong thing. PR #103 added
an `SdkMessage` variant for the SDK's top-level
`system` / `subtype: "permission_denied"`, with this reasoning in its own
comment:

> *"only THIS message reaches the top-level stream, which is exactly why
> suite-guard denials can never be confused with it here."*

**That assumption is false for the failure it was written to catch.**
Measured on `sabado-25`'s build session transcript:

- **8 denials, all `toolDenialKind: "user-rejected"`**
- every one delivered as a **`tool_result` with `is_error: true`** *inside
  the conversation* — never as a top-level system message
- **zero denial events in `runs/<runId>/journal.jsonl`** — the detector never
  fired

The commands that tripped it were ordinary work:

```
This Bash command contains multiple operations. The following parts require
approval: kill %1, pkill -f "vite preview --port 4599"

This Bash command contains multiple operations. The following parts require
approval: curl -s -o /tmp/health.out -w "STATUS:%{http...
```

The agent then received, three times:

> *"The user doesn't want to take this action right now. STOP what you are
> doing and wait for the user to tell you how to proceed."*

and stopped, correctly and explicitly:

> *"…means I don't have authorization to proceed with the build right now.
> I'm stopping here rather than retrying calls or working around the denials.
> No files have been changed in this turn."*

**Result: `build` ran 49.7s, recorded zero usage, wrote no artifact, and the
run blocked on the artifact message — the same symptom-not-cause report
`adw-bug-19` was supposed to end.**

- [ ] **R1 — ship `adw-bug-19` R1: an unattended run must not be able to
      request approval.** `acceptEdits` auto-approves edits but **not** Bash
      permission rules. Pin a mode that cannot prompt, or pre-approve the
      command surface. State the choice and its reason in `build.ts`'s doc
      comment beside the mode.
- [ ] **R1a — detect the denial where it actually arrives**: a `tool_result`
      with `is_error: true` carrying a `user-rejected` denial, not only the
      top-level `system`/`permission_denied` message. Keep the existing
      detector — add this path, do not replace it.
- [ ] **R1b — `suite-guard`'s denial must still NOT trip the failure path.**
      It is a deliberate, recoverable redirect. `adw-bug-19` R4 relied on the
      two being distinguishable by stream position; **that distinction is
      gone** once R1a reads tool_results, so the guard must now mark its own
      denials explicitly.
- [ ] **R1c — a run that stops on denials fails naming them**, with the tool
      and the rule, never with a missing-artifact message.
      *Amended 2026-09-27 by `adw-bug-24`: only the "user doesn't want to take
      this action" STOP text is fatal; "requires approval" rejections are retryable.*

---

## R2 — a ticket with no `attempts:` line parses, dispatches, and can never be finalized

Same run, second failure:

```
✗ sabado-25 could not be finalized — appendAttempt: no 'attempts:' field
  found in frontmatter; run …-1789996074209 recorded (journal only),
  ticket left as-is
```

The contract disagrees with itself:

| | |
|---|---|
| `src/intake/ticket.ts:314` | `attempts` is **optional, defaults `[]`** — so the ticket parses and dispatches |
| `src/intake/attempts.ts:63` | `appendAttempt` **throws** if the literal `attempts:` line is absent |

So a hand-written ticket without the line runs to a terminal outcome and then
**cannot be recorded**. `sabado-25` is stuck at `status: in-progress` with no
attempt history, which breaks the README's own invariant — *"no terminal path
leaves a ticket `in-progress`"* — and makes the ticket un-redispatchable
without a hand edit.

**Scope, measured:** 3 of 17 sabado tickets (`sabado-25`, `sabado-26`, plus
`README`) and 2 of 187 of our own lack the field. It is latent in every one
of them.

- [ ] **R2 — close the gap in one direction and say which.** Either
      `appendAttempt` creates the field when absent (preferred — the parser
      already treats it as optional, and `attempts:` is documented as
      *"written by the factory — never by hand"*), or `parseTicket` rejects a
      ticket lacking it before dispatch. **Not both, and not neither.**
- [ ] **R2a — a finalization failure must not be silent-ish.** Today the run
      exits non-zero with the ticket left mid-flight. Whatever R2 chooses,
      finalization must either succeed or leave the ticket in a state a
      re-dispatch can pick up.
- [ ] **R2b — no hand-repair of existing tickets as the fix.** Backfilling
      the five files is fine as cleanup, but it is not the requirement.

## Verify

- [ ] Red test first (Art. I) for R1a: a fixture whose agent receives a
      `user-rejected` `tool_result` currently ends with a missing-artifact
      failure; RED until it ends naming the denial.
- [ ] Red test for R1b: a `suite-guard` denial still lets the node continue
      to green.
- [ ] Red test for R2: a ticket without `attempts:` reaches a terminal
      outcome and is finalized (or is refused pre-dispatch — whichever R2
      chose), never left `in-progress`.
- [ ] **Live:** re-dispatch `sabado-25` and confirm `build` either completes
      or fails naming the denial — never a silent 50s zero-usage node.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- The `.adw/artifacts/` provision-time precondition (`adw-cost-00` R3a).
  Still worth having as defence in depth; still not the cause.
- Changing what any agent is asked to do. In both observed runs the agent
  behaved correctly given what it was told.
