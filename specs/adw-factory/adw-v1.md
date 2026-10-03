# Spec: adw-factory v1 — the chore lane, end to end

> Phase 1 — Specify. WHAT & WHY only. Technology choices live in `adw-v1-plan.md`.

- **ID:** adw-v1
- **Status:** ready
- **Author / date:** Silou + Claude, 2026-07-14
- **Origin:** 14-decision grilling session (2026-07-14), sourced from IndyDevDan's
  "software factory" ADW talk, Stripe's Minions posts, and alexop's factory essay.

## 1. Overview

adw-factory is a personal software factory: a standalone system that takes a typed
work ticket at one end and produces a reviewed-and-mergeable pull request at the
other, with **humans only at the two ends** (writing the ticket, reviewing the PR)
and **agents + deterministic code in the middle**. It exists because the
highest-leverage engineering work is now meta-engineering — building the system
that builds the system — and because owning the harness locally, on subscription
economics, compounds across every future project. Its first customer is cLens
(`/Users/silouane/agent-observability-project`), which is also its observability
provider: the factory is instrumented by the very product it builds.

V1 ships **one lane — chore — fully trustworthy**, proving every piece of shared
plumbing (intake, isolation, build/repair loop, gates, PR lifecycle, status
transitions, three-layer observability) before any lane multiplication.

## 2. Goals / non-goals

**Goals**

- A chore ticket dropped in the target repo travels unattended to a real,
  CI-green-or-explained pull request that a human merges.
- Every automated decision that can be anticipated is executed by deterministic
  code; agents act only where judgment is required.
- Every run is fully reconstructable after the fact from its journal, its trace,
  and its captured agent sessions.
- Isolation strength is an operator choice per run: lightweight local, fully
  contained local, or remote sandbox.
- Zero marginal dollar cost: runs on the existing subscription; the budget is
  rate-limit headroom, governed by bounded loops and a thrifty default model.
- The shakedown cruise produces real value: the first tickets are genuine cLens
  chores, not synthetic tests.

**Non-goals (v1)**

- Bug, feature, and hotfix lanes; routing beyond a typed field; triage/scout/plan
  agents.
- A router agent, a watcher daemon, scheduled runs, **a queue drainer or
  scheduler**.
  <br>*Amended 2026-09-11 (operator-approved), narrowing "parallel ticket
  execution" → "a queue drainer or scheduler":* the operator remains the
  scheduler (plan §2 decision 11), but two **operator-initiated** runs against
  **different** tickets may execute concurrently. The original constraint was
  never about execution safety — it was about not building autonomy before
  trust — and concurrency of manual dispatches costs no autonomy. It does cost
  a correctness guarantee at the shared git edge, which
  `adw-par-01-repo-commit-lock` pays for. Same-ticket concurrency stays
  forbidden (E7, unchanged).
- Auto-merge, auto-approval, or any factory-initiated write to a protected branch.
- Dashboards or new visualization surfaces (the data is banked from run #1; views
  come later).
- Search-based or agent-based context prefetch (the journal must first prove
  discovery is a bottleneck).
- Multi-user or hosted operation; API-key billing.
- Modifying cLens's published npm package to carry any factory code.

## 3. User stories

### Story 1 — Submit a chore, receive a PR

*As the operator, I want to write a small typed ticket and get back a finished
pull request, so that mechanical work costs me a ticket's worth of writing and a
review's worth of attention.*

**Acceptance criteria**

- When the operator starts a run, the factory shall select the named ticket, or
  else the highest-priority queued chore ticket in the target repo.
- When a ticket is selected, the factory shall validate its structure against the
  ticket contract before any workspace is provisioned or any agent runs.
- If a ticket is malformed (missing required fields, unknown type, no registered
  lane), then the factory shall reject it deterministically, listing every missing
  or invalid field, and shall consume no agent tokens doing so.
- When a valid chore ticket is dispatched, the factory shall mark it in-progress,
  provision an isolated workspace, and run the build agent with a
  deterministically assembled prompt containing the ticket body, the target's
  conventions, and pointers to the target's context documents.
- When all local gates pass, the factory shall push the work as a branch and open
  a ready-for-review pull request whose description carries the ticket link, a
  change summary, gate results, repair rounds used, and total duration.
- When the pull request opens, the factory shall mark the ticket in-review.

### Story 2 — Bounded self-repair

*As the operator, I want the factory to fix its own failures within hard limits,
so that transient mistakes never reach me but pathological runs never burn my
budget.*

**Acceptance criteria**

- When a build attempt completes, the factory shall run the target's local gates
  (lint, typecheck, tests) in order, as code, with no agent involvement.
- If a local gate fails, then the factory shall return a structured failure
  report to the same agent session (preserving its working context) and re-run
  the gates after the repair attempt.
- Where the target declares a `fix` command (a safe auto-fixer such as
  `biome check --write`, never `--unsafe`), the factory shall run it as code
  immediately before each gates pass that follows an agent node — after build
  and after each repair round — and never before the baseline, which measures
  the base as-is. The fix command's exit code shall never gate (lint still
  decides): a non-zero exit is journaled as a `fix-step` event and the gates
  run regardless. Formatting is a function, not an agent (Art. III), so no
  agent turn is spent on it.
- While a ticket is running, the factory shall permit at most 3 local repair
  rounds; on the 4th local failure it shall stop, mark the ticket blocked, and
  attach the full transcript and failure history.
- When remote CI fails on the opened pull request, the factory shall feed the CI
  log back for at most 1 repair round; a second CI failure shall block the ticket
  the same way.
- While any agent node is running, the factory shall enforce wall-clock and
  turn ceilings; v1 defaults shall be set high enough that repair rounds are the
  effective governor, and both ceilings shall be tunable per ticket.
- A *turn* is a top-level agent turn as streamed by the SDK: messages
  belonging to subagents (`parent_tool_use_id` set) are excluded from
  mid-stream metering; boundary accounting uses the SDK result's
  `num_turns`, which is message-scale (observed: 261 for a heavy clean
  run), and cap defaults shall be sized to that scale. *(Amended
  2026-07-15, M6 run-4 finding, operator-approved.)*
- If any ceiling is breached or the operator interrupts, then the factory shall
  abort gracefully: the ticket becomes blocked, all observability artifacts are
  flushed, and the workspace is kept intact for autopsy. *(Amended 2026-09-12,
  ticket `adw-caps-01-ceiling-deterministic-tail`, operator-approved: a
  **turn**-ceiling breach does not itself abort — the engine continues through
  any remaining **deterministic** nodes and refuses only the next node that
  would spend agent turns, so a run that finishes its deterministic tail after
  the breach reaches its real outcome instead of `blocked`. A **wall-clock**
  deadline or an operator interrupt still aborts unconditionally, at the next
  boundary, regardless of node kind — this amendment narrows the turn branch
  only.)*
- The factory shall never push work for which any local gate is failing.

### Story 3 — Human review gate

*As the operator, I want to be the only merger, so that accountability stays
human while the factory earns its track record.*

**Acceptance criteria**

- The factory shall never merge, approve, or close its own pull requests.
- The factory shall keep its own open pull request rebased onto its base branch
  (force-pushing only that attempt branch, with a lease); the operator merges
  what is already mergeable.
- When a factory pull request has been merged, the factory shall (on its next
  run) transition the ticket to done.
- When a factory pull request has been closed without merging, the factory shall
  transition the ticket to rejected with the reviewer's comments appended to
  the ticket body as context for a future run; re-entry to queued is an
  operator edit, never automatic.

### Story 4 — Operator-chosen isolation

*As the operator, I want to pick the isolation strength per run, so that simple
chores stay frictionless while risky or parallel work gets full containment.*

**Acceptance criteria**

- When starting a run, the operator shall be able to select one of three
  workspace kinds: local worktree, local container, or remote sandbox.
- While running under any workspace kind, agent nodes shall read and write only
  within the provisioned workspace.
- The workspace lifecycle (provision → execute → collect results → teardown)
  shall be identical across kinds from the pipeline's point of view.
- When a run ends in blocked or aborted state, the factory shall keep the
  workspace; a dedicated clean command shall be the only reclamation path.
- V1 shall ship the worktree kind fully working; container and remote kinds
  shall be delivered next, in that order, behind the same interface.

### Story 5 — Three-layer observability

*As the operator (and as cLens's author), I want every run observable at the
pipeline, agent, and standards layer, so that I can debug the factory, dogfood
cLens on it, and remain free to point any tracing backend at it later.*

**Acceptance criteria**

- When a run executes, the factory shall append a structured journal recording
  every node transition, gate result, repair round, token usage, wall time, and
  final outcome, keyed by ticket id.
- When an agent node runs, the factory shall inject cLens capture so the full
  session is recorded and tagged with the ticket id, ready for distillation.
- When a run executes, the factory shall emit one trace per ticket run in an
  open, vendor-neutral format — every pipeline node a span, agent spans parented
  under the run even when the agent executes in a container or remote sandbox —
  persisted locally and ingestible by any standard tracing backend without code
  changes.
- When a pull request is opened, its description shall quote the run's headline
  stats from the journal.

### Story 6 — Serial, watched operation

*As the operator, I want to be the scheduler in v1, so that every early run is
observed and trust is built on evidence.*

**Acceptance criteria**

- When invoked, the factory shall process exactly one ticket per invocation,
  streaming node-by-node progress to the terminal.
- While a ticket is in-progress or in-review, the factory shall refuse to start
  a second run on the same ticket.
- *PROPOSED amendment (ticket `adw-resume-01-continue-a-blocked-run-9ae0cb`;
  **not in force until the operator approves it** — approval is recorded by
  replacing "PROPOSED" with "operator-approved <date>"):* `blocked` is terminal
  **for a run**, not for the ticket. The operator may continue a blocked ticket
  with `adw resume <runId>`, which starts a **new run** linked to the blocked
  one by `resumedFrom`, re-enters the lane at the node that failed, in the same
  workspace, and moves the ticket `blocked → in-progress` in one commit. Resume
  is operator-initiated only; the factory never resumes a run by itself. A run
  that ended `green` or `in-review` cannot be resumed. Dispatch (E7) still
  requires `queued`; `blocked → in-progress` is legal only through the resume
  command, which takes the same lockfile CAS as a dispatch and refuses while the
  old run has no `run-end` and its process is alive. Same-ticket concurrency
  stays forbidden.
- The build agent's default model shall be a mid-tier workhorse; the model shall
  be declarable per pipeline node in lane configuration and overridable per
  ticket.

### Story 7 — Shakedown on real work

*As the operator, I want the first tickets to be genuine cLens chores, so that
proving the factory also improves the product.*

**Acceptance criteria**

- Before v1 is declared done, at least 3 real cLens chore tickets shall have
  traveled the full lane and been merged by the operator.
- The shakedown tickets shall be drawn from cLens's actual backlog (candidate
  pool at spec time: repo hygiene such as ignoring generated `coverage/` and
  `logs/` artifacts, outstanding lint/typecheck warnings, dependency bumps —
  final pick happens at shakedown, from what the repo needs then).

## 4. Non-functional requirements

- The factory shall incur zero marginal dollar cost in v1: all agent calls run
  on the operator's existing subscription authentication.
- A typical chore ticket shall complete (ticket selection → PR open) in under
  30 minutes of wall time.
- Ticket state shall live in exactly one place — the ticket file in the target
  repo — and every state transition shall be committed, making git history the
  audit log.
- The factory shall support any target repo that provides a thin configuration
  (location, gate commands, branch conventions); nothing cLens-specific may live
  in the pipeline itself.
- All factory prompts shall be versioned files in the factory repo, so prompt
  tuning is a reviewable diff.
- A failed or crashed run shall never leave the target repo's primary working
  copy or default branch modified, with one explicit exception: ticket status
  commits, which touch only `tickets/<id>.md` and are themselves the audit
  log.

## 5. Edge cases & error handling

- If the target repo has uncommitted changes in its primary working copy, then
  the factory shall proceed regardless (work happens in isolated workspaces);
  if the *ticket file itself* has uncommitted edits, the factory shall refuse
  dispatch until they are committed.
- If a branch for the ticket already exists (a previous attempt), then the
  factory shall create a fresh attempt branch with an incremented suffix rather
  than reusing or force-pushing.
- If a gate command itself fails to execute (missing script, misconfigured
  target), then the factory shall distinguish this from a gate *failure*, mark
  the ticket blocked as a target-configuration error, and not charge a repair
  round.
- If the subscription rate limit is exhausted mid-run, then the factory shall
  abort gracefully into blocked state with the workspace kept, and the journal
  shall name rate-limiting as the cause.
- If workspace provisioning fails, then the factory shall mark the ticket
  blocked without consuming any agent tokens.
- If the pull request cannot be opened (network, auth), then the factory shall
  retry the push/PR step deterministically a bounded number of times, then block
  with the local branch intact and named in the journal.
- If the review↻fix loop exhausts its rounds while a blocking finding
  remains, then the factory shall NOT block: gates are green by definition
  at that point, so the run proceeds to `commit → push → open-pr` carrying
  the unresolved findings, and the PR body states plainly that the review
  loop exhausted its rounds. `green` means "the PR is open" — which may
  include unresolved advisory findings — not "a reviewer approved it".
  (`specs/adw-v1.13-run-economics.md` §3 S4, §5 D4.) Every other retry loop
  (gates↻repair, CI-repair, watchdog trips, hard stops, turn/wall-clock
  ceilings) keeps blocking exactly as before — this amends only the one
  terminal path named above.
- If an agent session's tool call is denied for lack of human approval
  (headless), then the factory shall distinguish three outcomes: the STOP
  denial ("The user doesn't want to take this action right now…") is fatal to
  the node; a "requires approval" rejection is retryable by the model; and a
  tool-approval *outage* (the auto-mode classifier is down: "<model> is
  temporarily unavailable, so auto mode cannot determine the safety of
  <tool>…") is a transient for the engine, never the model's to work around.
  One outage denial is tolerated; at three in one session the node ends as a
  transient (retried within the node's transient-retry rounds, workspace left
  untouched) because beyond that the model starts working blind. When retries
  are exhausted the run blocks with a reason that names the outage and an
  `approval-outage` journal event, never "exhausted N repair rounds".
- If two invocations race on the same tickets directory, then ticket status
  transitions shall be atomic enough that a ticket is never dispatched twice.
- If remote CI is flaky (fails, then passes on re-run without changes), the one
  CI repair round absorbs it; the factory shall not distinguish flake from real
  failure in v1.

## 6. Success metrics

- ≥ 3 real cLens chores merged with zero human-written code — only ticket
  authoring and PR review.
- After shakedown, ≥ 80% of dispatched chore tickets reach in-review without
  blocking.
- 100% of runs (including blocked and aborted ones) have a complete journal, a
  complete trace, and tagged captured sessions — verified, not assumed.
- Median ticket → PR wall time under 30 minutes; repair rounds median ≤ 1.
- The operator can answer "what did run X do and why" from artifacts alone,
  without rerunning anything.

## 7. Open questions

- None blocking. Two deliberate deferrals: the product name (`adw-factory` is a
  working title, swap when a better one is probed) and grep-based context
  prefetch (adopt only if journals show discovery is a real cost).

## 8. Change log

- [ADDED] Everything — founding spec.
- [CLARIFIED] 2026-10-03 (adw-gates-11): Story 2 / S2.1 — "stop at the first
  introduced failure" becomes "route on the first introduced failure; report
  every gate". Later gates still run, report-only, marked `after-failure` in the
  gate summary and the repair prompt; they never change routing. A gate may opt
  out with `reportOnlyAfterFailure: false` (see `adw-v1.1-lanes.md`).
- [CLARIFIED] 2026-07-15 (M6 shakedown, operator-approved): Story 2 turn
  semantics — a turn is a top-level agent turn; subagent messages excluded
  from mid-stream metering; boundary accounting is SDK `num_turns`
  (message-scale) and cap defaults size to it.
- [AMENDED] 2026-07-14, post-review: Story 3 — a closed-without-merge PR leaves
  the ticket `rejected` with comments attached; operator edit is the only
  re-entry to `queued` (was an ambiguous "reopen", contradicting the plan).
- [AMENDED] 2026-07-14, post-review: crash-safety NFR — ticket status commits
  on the default branch are the explicit exception to "never modified".
- [AMENDED] 2026-07-14, operator gate on adw-m1-03: E1's dirty-ticket-file
  refusal is generalized from dispatch to every status transition — a mid-run
  operator edit to the ticket file must never be silently swept into a
  factory status commit.
- [AMENDED] 2026-09-12, ticket `adw-caps-01-ceiling-deterministic-tail`,
  operator-approved: Story 2's "abort gracefully" clause narrows on a turn
  breach — deterministic nodes still run and the run reaches its real outcome;
  only the next agent node is refused. Wall-clock and operator-interrupt
  aborts are unchanged.
- [AMENDED] 2026-09-19, ticket `adw-review-04-a-non-converged-review-opens-a-pr`,
  spec `adw-v1.13-run-economics.md` §3 S4/§5 D4: §5's `green | blocked`
  terminal contract is no longer "a reviewer approved it" — a review↻fix
  loop that exhausts its rounds with a blocking finding still outstanding now
  reaches `green`, carrying the unresolved findings into the PR rather than
  discarding the run. Every other exhaustion path is unchanged.
- [AMENDED] 2026-09-29, ticket `adw-merge-01-a-pr-that-falls-behind-its-base-is-rebased-by-the-factory`,
  pending operator approval: Story 3 gains a rebase bullet — the factory
  rebases its own attempt branch onto its base (before opening the PR, and
  while it is open) and force-pushes only that branch, with a lease. This is a
  new write to the remote after `in-review`; it is not a merge, approval or
  close, and Story 3's "never merge, approve, or close" stands.
- [AMENDED] 2026-10-02, ticket `adw-bug-40`: §5 gains the headless
  tool-approval denial rule (STOP fatal / "requires approval" model-retryable /
  approval outage engine-transient at 3 denials; the blocked reason names the
  outage).
