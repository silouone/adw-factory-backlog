---
id: adw-m1-09-agent-nodes
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m1-05-engine, adw-m1-06-worktree-workspace, adw-m1-07-target-config-prompt]
attempts: []
---
# Agent nodes — build + repair (same session)

## Context

The only two agent node types in v1 (plan §2 gate III). Thin edges over
`@anthropic-ai/claude-agent-sdk`'s `query()`; all logic that can be
deterministic stays in the engine. Repair resumes the **same session** so
working context is preserved (decision 7). Tests fake the SDK boundary —
no tokens are spent before adw-m1-11.

## Deliverables

- `test/pipeline/nodes/agents.test.ts` (first, red — injected fake `query`)
- `src/pipeline/nodes/build.ts`, `src/pipeline/nodes/repair.ts`

## Requirements

- [x] `query` is injected (constructor/param), production default = SDK —
      the fake asserts exact options (Art. IX testability)
- [x] Build node calls `query` with: `cwd` = workspace path; `model` from
      lane node config, overridden by ticket `model:` (S6.3); permission
      mode / allowed tools scoped to the workspace (S4.2, Art. VII); `env`
      from `workspace.agentEnv()` (S1.4)
- [x] Captures `sessionId` from the SDK init message; accumulates turns +
      token usage; returns them for the engine's `node-end` journal event
      (S5.1)
- [x] Repair node calls `query({ resume: sessionId })` with a structured
      failure-report prompt (from the gates `GateFailureReport`, incl.
      diff-so-far) — same session, rounds counted by the engine, ≤3 (S2.2,
      S2.3)
- [x] Turn ceiling from ctx enforced by aborting the stream when exceeded
      (S2.5)
- [x] SDK rate-limit exhaustion maps to graceful abort with the journal
      naming rate-limiting as the cause (E4)
- [x] Agent output summary (final message) is retained in ctx — the PR body
      needs an agent-written summary in M2 (S1.5)

Note (operator-approved contract, 2026-07-14): `permissionMode:
"acceptEdits"` is the workspace scoping mechanism. Error policy is split:
rate-limit → graceful fail naming the cause (E4); any other stream error is
rethrown for engine wrapping (Art. IX). The gate-failure report shape
`{gate, cmd, exitCode, stdoutTail, stderrTail, diffSoFar}` is pinned by
this ticket's tests — adw-m1-08 must conform to it.

## Build protocol (Art. I)

1. Fake `query` yielding scripted SDK message streams (init w/ sessionId,
   turns, usage, result; a rate-limit error variant). Tests per requirement.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green (no network,
no tokens).

## Out of scope

Live agent runs (adw-m1-11), cLens capture hooks (adw-m3-04).
