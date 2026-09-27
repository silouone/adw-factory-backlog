---
id: adw-m3-04-clens-capture
type: feat
status: done
priority: 2
created: 2026-07-14
epic: adw-m3
depends: [adw-m2]
attempts: []
---
# cLens capture injection + ticket tagging

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`. Risky set (handoff): the live-injection
> seam, plan §6 risk — propose operator red-test review. Independent of
> m3-01/02/03 — can run in parallel from epic start.

## Context

S5.2: every agent session captured by cLens and tagged with the ticket id,
ready for distillation. **Why workspace files cannot work (plan §6 risk,
verified in the cLens source):** `clens init` installs its 17
`clens-hook <event>` entries into `.claude/settings.local.json` and writes
sessions to `<projectDir>/.clens/sessions/*.jsonl` — both uncommitted, so
neither exists in a fresh worktree. Injection therefore rides SDK hook
options at the injected `query()` seam. cLens lives at
`/Users/silouane/agent-observability-project` (hook entry:
`clens-hook <eventType>`, JSON payload on stdin — Claude Code's hook
protocol).

## Deliverables

- `test/observability/clens.test.ts` + node-wiring tests (first, red)
- `src/observability/clens.ts`
- `src/pipeline/nodes/build.ts` (`AgentQueryOptions` hooks slot — pins
  updated red-first) · `src/live-query.ts` (map onto SDK `options.hooks`)
- `src/pipeline/nodes/open-pr.ts` `renderSessionIds` (+ snapshot)
- journal: a `capture` record shape (`src/observability/journal.ts`)

## Requirements

- [x] `src/observability/clens.ts` builds an SDK hooks option set that
      forwards each hook event payload to `clens-hook <event>` (stdin
      JSON), spawned with cwd/env chosen so sessions land under
      `runs/<runId>/.clens/sessions/` (amended: cLens hardcodes the
      `.clens/sessions` suffix — see Implementation notes) — a run
      artifact that survives `adw clean` of the workspace (metric 3) —
      and carry `ADW_TICKET_ID` (already in `agentEnv()`) so every
      captured session maps to its ticket (S5.2)
- [x] The injected `AgentQuery` seam gains an opaque hooks slot
      (`AgentQueryOptions` is pinned by tests — extend red-first);
      `live-query.ts` maps it onto the SDK's hook options; fake-query
      tests assert the wiring on build, repair AND ci-repair
- [x] Capture is middleware: injection disabled → no hooks passed, zero
      behavior change; a failing/missing `clens-hook` binary is journaled
      and NEVER fails the agent node (the run is flagged incomplete, not
      blocked — same tolerance stance as m3-01 tracing)
- [x] Captured cLens session ids (amended: = the SDK session id —
      cLens keys its JSONL by the payload `session_id`, source-verified;
      this equality is the m3-05 join key) are journaled — a `capture`
      event carrying sessionId + path, so the journal remains the one
      run history (Art. VIII) and m3-05 can cross-check
- [x] `renderSessionIds` fills the PR-body slot left empty by adw-m2-02
      from the run's journal records (S5.4); zero captures renders an
      explicit "none captured" line, never a silently-empty slot (N5)
- [x] Workspace hygiene: if any capture side-file lands in the workspace,
      extend the commit/ci-commit sweep exclusions
      (`':(exclude).claude'` pattern, push.ts + ci-round.ts — M2 live
      finding) so nothing reaches the PR; decide and pin with a test
- [x] **LIVE check is part of this ticket** (real tokens — explicit
      operator go-ahead first, as in M2): one real agent run with capture
      confirmed — session JSONL present under the run dir, ticket-tagged,
      readable by cLens tooling (`clens distill` or equivalent).
      **Done 2026-07-15** on run `m3s-001-1784104068186` (operator-approved
      live run): session `32628fbd…` captured to
      `runs/<runId>/.clens/sessions/`, journal `capture` record `ok:true`,
      ticket id throughout the payloads, `clens list` + `clens distill`
      both read it; PR #4's body renders the session id (adw-m3-05 has the
      full audit)

## Implementation notes (green 2026-07-15, adw-m3-04)

Two source-verified reconciliations vs. the requirement text (cLens source at
`/Users/silouane/agent-observability-project`), pinned by tests:

- **Sink path.** Sessions land at `runs/<runId>/.clens/sessions/`, not the
  literal `runs/<runId>/sessions/`: cLens's hook hardcodes the `.clens/sessions`
  suffix and resolves its root by walking up from the agent cwd to the nearest
  `.clens/` (`resolveProjectRoot`). We pre-seed `runs/<runId>/.clens/` above the
  workspace so the walk-up lands there. Still a run artifact surviving workspace
  clean (metric 3).
- **Captured id.** cLens keys its JSONL by the payload `session_id`, which IS the
  SDK session id — so `capture.sessionId` EQUALS the build node's
  `usage.sessionId` (contra "≠ SDK session ids"). This equality is exactly the
  join key m3-05 relies on.
- **Sweep decision (req. 6).** No extension: captures land in a workspace
  SIBLING (`runs/<runId>/.clens/`), never inside the workspace, so the `.claude`
  sweep in push.ts/ci-round.ts is untouched. Pinned by the "sink is OUTSIDE the
  workspace" test.
- **Live wiring is orchestrator-owned.** Everything ships inert (disabled → zero
  behavior change); `makeClensHooks`/`liveClensSpawn` are constructed and wired
  into the lane by the CLI orchestrator (cli.ts/chore.ts), enabling the LIVE
  check.

## Build protocol (Art. I)

1. Red tests: hook-set construction (event → clens-hook invocation), seam
   wiring on all three agent nodes, failure tolerance, journal record, PR
   body render (+ empty case). **Operator reviews the red set** (risky
   ticket) unless delegated to a validator.
2. Implement to green; live check last, after operator go-ahead.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green; live check
shows a captured, ticket-tagged session; PR body renders session ids.

## Out of scope

Distillation/analysis of captured sessions (cLens's job); tracing spans
(m3-01/02/03); modifying cLens itself (spec non-goal).
