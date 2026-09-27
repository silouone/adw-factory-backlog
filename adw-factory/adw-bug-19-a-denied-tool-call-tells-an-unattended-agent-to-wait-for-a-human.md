---
id: adw-bug-19-a-denied-tool-call-tells-an-unattended-agent-to-wait-for-a-human
type: bug
status: done
priority: 1
review: false
created: 2026-09-21
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-bug-19-a-denied-tool-call-tells-an-unattended-agent-to-wait-for-a-human-1789973316339","branch":"adw/adw-bug-19-a-denied-tool-call-tells-an-unattended-agent-to-wait-for-a-human","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-19-a-denied-tool-call-tells-an-unattended-agent-to-wait-for-a-human-1789973316339/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/103","provider":"claude","model":"sonnet"}]
---
# A denied tool call tells an unattended agent to wait for a human who does not exist

> **Root cause of the `test`-artifact failure class** — 4 sabado runs
> (**$113.25**) plus `adw-cost-01` (233 turns of `plan`+`build`, 50 minutes,
> discarded). `adw-cost-00` R3 investigated this and concluded "a
> **model-behavior** tail risk, not a deterministic bug", then shipped neither
> R3 nor R3a. **That conclusion is wrong and this ticket supersedes it.**
> The model behaved correctly. It was told to stop and wait.

## The chain, reproduced on our own repo

Run `adw-cost-01-bound-what-a-tool-call-may-inject-1789953612869`, node
`test`: **23.0 seconds, zero usage recorded, no artifact, run blocked.**

Read from the session transcript, in order:

1. **`permissionMode` is `"acceptEdits"`** — pinned at
   `src/pipeline/nodes/build.ts:823`, and typed as that literal at `:118`.
   Observed on **26 of 26** agent records across the last 12 runs.
2. The agent ran the full suite. **`suite-guard` denied it** (correctly —
   `adw-perf-04`), with its helpful message pointing at `bun run test:unit`.
3. The agent then issued compound Bash commands (`jobs`, `grep`, `find`).
   Under `acceptEdits` these **require approval**:
   `toolDenialKind: "permission-rule"` —
   *"This Bash command contains multiple operations. The following part
   requires approval: jobs"*.
4. **There is no human in an unattended factory run.** The calls resolved to
   `toolDenialKind: "user-rejected"` — **5 of them** — each carrying:

   > *"The user doesn't want to take this action right now. **STOP what you
   > are doing and wait for the user to tell you how to proceed.**"*

5. The agent obeyed, exactly as instructed:

   > *"Understood — stopping here and waiting, as instructed. I won't take
   > further actions until told how to proceed."*

6. Node ended with no `.adw/artifacts/test.md`. The engine reported the
   **symptom** — a missing artifact — and the run blocked, discarding a
   completed 183-turn `build`.

**`.adw/artifacts/` existed and was writable** (it held `plan.md` and
`build.md` from the same run). The missing-directory theory is dead.

## Why it is silent and therefore expensive

The node burns ~23s, records **zero usage**, and fails on a message that
names the artifact rather than the denial. Every post-mortem so far has
chased the artifact. Nothing in the journal says a tool call was ever denied.

## Requirements

- [ ] **R1 — an unattended run must never be able to request approval.**
      Either pin a mode that cannot (`bypassPermissions`) for factory-driven
      sessions, or pre-approve the command surface the agent is allowed.
      **`acceptEdits` auto-approves edits but not Bash permission rules**,
      which is the gap. Whatever is chosen must be stated in
      `build.ts`'s doc comment beside the mode, with the reason.
- [ ] **R2 — a denial is a node-level event, not a silent turn.** When a tool
      call is denied, the engine journals it (tool, kind, reason). "Why did
      this node do nothing" must be answerable from
      `runs/<runId>/journal.jsonl` alone (Art. VI).
- [ ] **R3 — a `user-rejected` denial in an unattended run fails the node
      immediately**, with a reason naming the denied tool and the rule. It
      must not leave the agent waiting and be reported later as a missing
      artifact. **Fail fast with the real cause** (Art. IX).
- [ ] **R4 — `suite-guard`'s own denial must keep working and must NOT trip
      R3.** It is a deliberate, recoverable redirect ("use `bun run
      test:unit` instead"), not a dead end. R3 targets denials with no
      recovery path. If the two cannot be told apart by `toolDenialKind`,
      the guard must mark its own denials explicitly.
- [ ] **R5 — no change to what any stage is asked to do.** No lane, prompt or
      artifact-contract change.

## Verify

- [ ] Red test first (Art. I): a fixture whose agent receives a
      `user-rejected` denial currently ends with a missing-artifact failure;
      RED until it ends with a denial-named failure.
- [ ] Red test: a `suite-guard` denial still lets the node continue and
      reach green (R4's guard against over-firing).
- [ ] Red test: a denial appears in the journal with its tool and kind.
- [ ] **Live:** re-dispatch `adw-cost-01` and confirm the `test` node either
      completes or fails with a reason naming the denial — never with a
      silent 23-second zero-usage node.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- The `.adw/artifacts/` provision-time precondition (`adw-cost-00` R3a). It
  is still worth having as defence in depth, but it is **not** the cause and
  would not have prevented any observed failure.
- Anything about what the agent should have run instead. The agent's
  behaviour was correct given the message it received.
