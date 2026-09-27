---
id: adw-perf-05-steps-are-isolated-and-the-plan-is-the-contract
type: feat
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# Steps are isolated: the feat lane hands off through `plan.md`, never through a shared session

> **Operator, 2026-09-27:** *"for step hygiene, we want them isolated, the
> contract is the spec file, in this case the plan.md."*
>
> This **reverses** the session-resume choice made by `adw-perf-01` (done). Its
> checklist offered "`build` resumes `plan`'s session **or** the plan artifact
> is made rich enough that build does not re-explore". It picked resume. This
> ticket picks the artifact, on the evidence below.

## Evidence: run `adw-store-01-tickets-dir-1790448708493` (2026-09-26)

- `build` **resumed** `plan`'s session (`feat.ts`,
  `resumesSession: { from: "plan" }`). `prompts/feature-build.md` **still
  inlines** `{{ticketBody}}` and `{{plan}}`. So the build paid for the ticket
  and the plan three times: in plan's prompt, as plan's own `Write` of
  `plan.md`, and again inlined. It also inherited all of plan's exploration.
  **The build's first call was already 183k tokens**, before a single edit.
- Resume did not buy what perf-01 wanted. The build **still re-explored**:
  78 source `Read`s, 43 test-file `Read`s, and most of 116 other Bash calls
  were `sed`/`grep` reads.
- The context then grew from 183k to 619k (no hop, see `adw-bug-23`). Generation
  slowed from 61 tok/s (plan) to 42 tok/s (build). 435 calls read 176 M cached
  tokens, **~$63–121**, and the node died at 91 min (see `adw-bug-22`).
- The plan step wrote a **44.5 KB `plan.md`** from an **18.4 KB ticket**.

## The rule this ticket establishes

**Each agent step is its own fresh session. The only thing that crosses a step
boundary is a file in `.adw/artifacts/` (plus the ticket and the repo on
disk).** No step inherits another step's conversation. For the feat lane:

| step | session | receives | must produce |
|---|---|---|---|
| `plan` | fresh | ticket + conventions + context pointers | `plan.md`, self-sufficient for building |
| `build` | **fresh** | ticket + **`plan.md`** + conventions | code changes + `build.md` |
| `test` | **fresh** | ticket + `plan.md` + `build.md` + the working-tree diff | tests + `test.md` (or today's artifact) |

## Requirements

- [x] **R1 — no cross-step resume in the feat lane.** Remove
      `resumesSession: { from: "plan" }` (build) and `{ from: "build" }`
      (test) from `feat.ts`. Remove the resume-related comments that justify
      them, and replace them with one comment naming this ticket and the rule.
      A test asserts that the feat lane's `build` and `test` agent calls carry
      **no** `resume` option.
- [x] **R2 — the plan is a contract.** `prompts/feature-plan.md` requires a
      `plan.md` that a fresh session can build from without re-exploring. It
      must contain the files to touch and why, the order of steps, the tests
      per step, and the exact existing symbols and signatures the build must
      call. It also carries a **size budget**: the plan should not exceed the
      ticket body's length by more than 1.5×, and must never restate the
      ticket.
- [x] **R3 — build receives the contract, once.** `prompts/feature-build.md`
      keeps `{{ticketBody}}` and `{{plan}}`, which is now correct because the
      session is fresh. It tells the agent that the plan is the contract: read
      only the files the plan names, and explore beyond them only when the
      plan is demonstrably wrong, recording that as a discrepancy in
      `build.md`.
- [x] **R4 — test receives the contract plus what build did.**
      `prompts/feature-test.md` gains `{{plan}}`, the build artifact and the
      working-tree diff as inputs, so it does not re-derive what was built. The
      diff input is size-bounded, with the same hunk manifest as `review-*`
      prompts (adw-cost-03).
      *Amended 2026-09-27: the review prompts no longer truncate `{{diff}}`;
      since adw-cost-03 they get a hunk manifest (headers + pointers to hunk
      files), so "the same truncation policy" named no existing policy.*
- [x] **R5 — artifacts are mandatory handoffs.** If `plan.md` is absent or
      empty, `build` fails before spending a token, with a reason naming the
      missing artifact. The same applies to `build.md` for `test`. (The
      "artifact absent → fail" behaviour exists for the writer; this makes the
      **reader** fail fast too.)
- [x] **R6 — tests.** Lane tests: fresh sessions (no `resume`) for build and
      test; the build prompt contains the plan text exactly once; the test
      prompt contains the plan, the build artifact and a bounded diff; a
      missing `plan.md` fails build with the descriptive reason and zero agent
      calls.
- [x] **R7 — measure.** On one live feat run after the change, record in the PR
      body the build's **first-call context** (target: under 40k tokens, versus
      183k) and the count of build `Read` calls on files the plan named versus
      files it did not.
      *Measured 2026-09-27 on `adw-cost-04-take-the-agent-out-of-the-loop-1790473998753` (PR #111 was already merged, so recorded here and in `ai_docs/2026-09-27-orchestration-log.md`): build first-call context **29.1k** (was 183k). The Read split was not measured.*

## Out of scope (see `adw-perf-06`)

The bug lane (`build-fix` resumes `build-test-only`) and the fix loops
(`repair` resumes `build`, `ci-repair` resumes the latest session) follow the
same rule in a separate ticket.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`, plus R7's live run.

## Blocked by

None. This ticket can start immediately. It pairs with
`adw-bug-23-the-turn-cap-never-counts-a-claude-turn`: isolated steps keep the
build's start small, and the hop keeps it small while the build runs.
