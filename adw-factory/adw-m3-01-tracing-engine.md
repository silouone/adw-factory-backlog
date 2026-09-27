---
id: adw-m3-01-tracing-engine
type: feat
status: done
priority: 2
created: 2026-07-14
epic: adw-m3
depends: [adw-m2]
attempts: []
---
# OTel tracing engine

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`. Orchestrator-owned (judgment-heavy).

## Context

Layer 3 of the observability trio (S5.3, decision 9). The engine's `runNode`
already brackets every node with `node-start`/`node-end` journal events —
the span middleware goes inside the same bracket, so a node cannot run
untraced (Art. VI). Tracing stays injected middleware with a no-op default:
pipeline logic never depends on it, and disabled mode is zero behavior
change (plan §2 watch item). Pattern lifted from translated-language-service
`~/coorp/translated-language-service/src/common/lib/tracing.ts`: OTel-native
provider + injected `SpanExporter` + no-op fallbacks that never break the
pipeline ("the pipeline must survive tracing failures").

## Deliverables

- `test/observability/tracing.test.ts` + engine-middleware tests (first, red)
- `src/observability/tracing.ts`
- `src/pipeline/engine.ts` middleware seam (`RunOptions.tracer`, no-op default)
- New prod deps (operator visibility): `@opentelemetry/api`,
  `@opentelemetry/sdk-trace-base`, `@opentelemetry/sdk-trace-node`,
  `@opentelemetry/context-async-hooks`, `@opentelemetry/core`,
  `@opentelemetry/resources`

## Requirements

- [x] `RunOptions` gains an optional injected `tracer`; absent → no-op:
      identical `RunOutcome`, byte-identical journal, zero new I/O — proven
      by an engine test running the same stub lane with and without (plan §2)
- [x] One trace per **runId**; root span opened at run-start, ended at
      run-end, carrying `adw.ticket_id`, `adw.run_id` and the outcome as
      attributes. The CI mini-lane (own ciRunId + own journal, adw-m2-04)
      gets its own trace by construction — the trace model mirrors the
      journal 1:1, one representation per concept (Art. VIII)
- [x] `traceId` is a pure, documented function of `runId` (hash); span ids
      come from an injectable generator — tests pin exact ids (Art. IX)
- [x] Child span per node, opened/closed in `runNode` exactly where
      `node-start`/`node-end` are journaled (Art. VI); the span records the
      routing outcome (`next|retry|fail`), and a thrown node ends its span
      with ERROR status + recorded exception. `round` and `abort` land as
      span events on the root span
- [x] Tracing failure (init, span open, export) warns and never fails the
      run — the TLS no-op-on-error pattern
- [x] Nodes stay tracing-unaware: no file under `src/pipeline/nodes/` or
      `src/pipeline/lanes/` imports tracing — enforced by a grep-guard test
      (mirrors the sync-pr-state negative-capability guard)

## Design decisions (approve at the refinement gate)

- OTel packages are used directly (Art. VIII), but the ENGINE sees only a
  minimal factory-owned `Tracer` interface defined in tracing.ts; OTel types
  never leak into engine signatures — that is what keeps the no-op default
  honest and the middleware removable.
- Trace-per-runId refines plan §5's "one trace per run" wording — no
  amendment needed: the journal already defines runId as the run.

## Build protocol (Art. I)

1. Red tests: no-op equivalence; expected span tree from a stub lane under
   `InMemorySpanExporter` (root + node children, retry/repair spans, error
   span, abort event); deterministic trace id; grep-guard.
2. Review gate (operator or validator per protocol) → implement to green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green; the stub-lane
memory-exporter test shows the expected span tree.

## Out of scope

Filesystem OTLP export + CLI wiring (adw-m3-02), TRACEPARENT + span typing
(adw-m3-03), cLens capture (adw-m3-04).
