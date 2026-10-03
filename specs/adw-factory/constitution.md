# adw-factory constitution

> Rarely-changing principles every spec and plan must honor. The Plan phase checks each
> feature against these as **gates**: pass, or justify the exception in writing.
> Amend deliberately, with rationale — this is the factory's architectural DNA.

## Article I — Test-first (non-negotiable)

No implementation code before: (1) tests written, (2) reviewed/approved, (3) confirmed
failing (red). Behavior is defined by tests before it exists. The factory is the system
that builds systems — its own correctness bar is the highest in the house.

## Article II — Simplicity / KISS-to-start

Prefer the smallest design that satisfies the spec. One lane until it earns trust.
No speculative generality: a capability is added when a journaled run demonstrates the
need, not when an article says it's coming.

## Article III — Code > engineers > agents

Every decision that can be anticipated is a deterministic code node — zero tokens, zero
hallucination, light speed. Agents are reserved for judgment under uncertainty
(implement, repair). A subtask migrates from agent to code the moment it becomes
predictable. Never an agent where a function suffices.

## Article IV — Humans at the two ends, never the middle

A human writes the ticket; a human merges the PR. Everything between is agents + code.
The factory never merges, never pushes to a protected branch, and never edits a ticket's
intent. An agent cannot be held accountable.

## Article V — Bounded autonomy

Every loop has a hard ceiling declared in code (repair rounds, CI rounds, time, turns).
Breaching a ceiling has exactly one outcome: stop, mark blocked, hand the human a
complete transcript. No unbounded retry exists anywhere in the system.

> **[AMENDED] 2026-09-19**, ticket `adw-review-04-a-non-converged-review-opens-a-pr`,
> spec `adw-v1.13-run-economics.md` §3 S4 / §5 D4: one ceiling — the review↻fix
> loop's round cap — now breaches to `next` instead of `blocked` when every gate
> the loop re-checks in-process is already green. The retry itself stays bounded:
> the loop still stops trying to converge at the same declared ceiling as before;
> only the terminal action taken once it stops changes. "Hand the human a complete
> transcript" is satisfied by the pull request body carrying the unresolved
> findings, not a blocked run, because it is the *review* that failed to converge,
> not the code under review. Every other ceiling in the system — gates↻repair,
> CI-repair, watchdog trips, hard stops, wall-clock and turn ceilings — is
> unchanged: breaching them still has exactly the original one outcome.

## Article VI — Observable by construction

No node runs without leaving a journal entry and a trace span. A run that cannot be
replayed from its artifacts (journal + spans + captured sessions) is a defective run,
even if its PR merged. The factory dogfoods the observability it was built to serve.

## Article VII — Isolation always

No agent node executes outside a provisioned Workspace (worktree, container, or remote
sandbox). Nothing is pushed before all local gates pass. Aborted workspaces are kept for
autopsy, reclaimed only by explicit clean.

## Article VIII — Anti-abstraction

Use the Agent SDK, git, and gh directly — no wrapper layers until two call sites demand
one. One representation per concept: the ticket file is the only ticket state, the
journal is the only run history.

## Article IX — Craft

Pure functions with side effects extracted to the edges; immutable structures; strict
TypeScript; fail fast with descriptive errors carrying ticket id + node name. The
factory's code reads like the code it is trusted to produce.
