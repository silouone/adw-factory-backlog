---
id: adw-m3-03-traceparent
type: feat
status: done
priority: 2
created: 2026-07-14
epic: adw-m3
depends: [adw-m3-01-tracing-engine, adw-m3-02-otlp-exporter]
attempts: []
---
# Trace-context propagation + span typing

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`. Risky set (handoff): cross-process context
> that M4/M5 container/remote inherit — propose operator red-test review.

## Context

S5.3: agent spans parent under the run even when the agent executes in a
container or remote sandbox — the W3C `traceparent` cross-process pattern
(TLS `withExtractedContext`). **Refinement finding (amendment rule):**
plan §5 places `TRACEPARENT` in `workspace.agentEnv()`, but the correct
parent is the *calling agent node's span* (build / repair / ci-repair),
which only the engine knows and which changes per node — a static
per-workspace env cannot carry it. Decision: the engine exposes the active
node's context on `RunContext`, and the agent nodes merge it over
`agentEnv()`. This keeps the workspace contract tracing-unaware, which is
exactly what makes propagation uniform across worktree/container/remote
kinds by construction (S4.3). ⚠ Includes a proposed plan §5 wording
amendment (Workspace contract comment) — approve at this gate, never
silently diverge.

## Deliverables

- Tests first (red): engine ctx propagation, env merge on all three agent
  nodes, simulated cross-process parent/child, span typing
- `src/pipeline/engine.ts`: `RunContext.traceparent?` (+ `EngineNode.spanType?`)
- `src/pipeline/nodes/build.ts`: `baseOptions` merge (repair + ci-repair
  inherit via the shared helper); `AgentQueryOptions` pins updated red-first
- `src/workspace/worktree.ts`: slot comment updated (no contract change)
- plan §5 amendment text (one sentence, applied on operator approval)

## Requirements

- [x] When tracing is enabled, `ctx.traceparent` is a valid W3C header
      `00-<traceId>-<activeNodeSpanId>-01` for every node; absent under the
      no-op tracer — asserted by engine tests with pinned ids (m3-01)
- [x] `baseOptions` merges `TRACEPARENT` into the agent env over
      `agentEnv()` when ctx carries it; build, repair AND ci-repair all
      pass it (fake-query tests assert the exact env, as today)
- [x] Cross-process proof: a test extracts the env's TRACEPARENT with the
      W3C propagator, opens a child span from it, and asserts same traceId
      + parent span id == the agent node's span id (the TLS pattern)
- [x] Span typing: agent nodes' spans carry type `AGENT`; deterministic
      nodes (dispatch, provision, gates, commit, push, open-pr, ci-commit)
      carry `TOOL` (plan §5) — via `EngineNode.spanType` consumed by the
      m3-01 middleware, defaulting to `TOOL`; recorded as a span attribute
      in the exported OTLP files
- [x] Workspace contract suite (`test/workspace/contract.ts`) unchanged —
      TRACEPARENT is deliberately NOT a workspace concern; `agentEnv()`
      keeps ADW_TICKET_ID (+ M4 auth slots)

## Build protocol (Art. I)

1. Red tests as above; **operator reviews the red set** (risky ticket —
   M4/M5 inherit this seam) unless the operator delegates to a validator.
2. Implement to green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green; one live
M1-style run (operator go-ahead, `env -u GITHUB_TOKEN -u GH_TOKEN` launch)
shows agent spans typed AGENT and parented under the run root in the
exported OTLP files.

## Out of scope

cLens capture (adw-m3-04); consuming TRACEPARENT inside agent processes
(future — v1 only guarantees the env is correct and extractable).
