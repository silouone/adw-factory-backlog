---
id: adw-m1-06-worktree-workspace
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m0-01-toolchain]
attempts: []
---
# Workspace contract + worktree implementation

## Context

Isolation always (Art. VII): no agent executes outside a provisioned
Workspace. The interface is the plan's one deliberate abstraction — three
real implementations planned (worktree M1, container M4, remote M5), so the
contract tests written here must be **generic over the interface** and reused
by M4/M5.

## Deliverables

- `test/workspace/contract.ts` — interface-generic contract test suite
- `test/workspace/worktree.test.ts` (first, red — temp git fixture repo)
- `src/workspace/types.ts` — exactly plan §5:
  `Workspace { kind, path, exec(cmd, opts?), agentEnv(), teardown() }`
- `src/workspace/worktree.ts`

## Requirements

- [x] `provisionWorktree(target, ticket, runId)`: branch
      `<branchPrefix><ticketId>` cut from `origin/<base>` when a remote
      exists, else local `<base>` — ticket status commits never ride attempt
      branches (plan §5 status-commit mechanics); `git worktree add` under
      `runs/<runId>/workspace`, then run `target.setup` inside it (S1.4)
- [x] Existing branch for the ticket → fresh attempt branch with incremented
      suffix (`-2`, `-3`, …); never reuse, never force-push (E2)
- [x] `exec(cmd)` runs with cwd = workspace path, returns
      `{code, stdout, stderr, durationMs}`; used for gates, git, setup
- [x] `agentEnv()` returns `ADW_TICKET_ID`; `TRACEPARENT` and auth slots
      documented, filled in M3/M4 (plan §5)
- [x] `teardown()` removes worktree + branch — invoked **only** by explicit
      clean; no engine failure path calls it (S4.4, Art. VII)
- [x] Provision failure → typed `WorkspaceError` carrying ticketId, before
      any agent runs — zero tokens (E5)
- [x] The target repo's primary working copy and default branch are never
      touched, verified even with uncommitted changes present there (N6, E1)
- [x] Lifecycle identical across kinds from the pipeline's viewpoint —
      encode as the reusable contract suite (S4.3, S4.5)

Note: baseDir is injected for testability (tests use temp dirs); production
callers pass the runs-root so workspaces land at `runs/<runId>/workspace` per
plan §5. The `origin/<base>` remote branch-point path is implemented
(`resolveBranchPoint`) but exercised only from M2 (fixtures have no remote).

## Build protocol (Art. I)

1. Fixture: temp git repo with a base branch and a fake `setup` script.
   Contract suite + worktree-specific tests (branch naming, suffix
   increment, dirty-primary-copy untouched). Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

`container.ts` (adw-m4-02), `e2b.ts` (adw-m5-02), `adw clean` CLI (adw-m2-05).
