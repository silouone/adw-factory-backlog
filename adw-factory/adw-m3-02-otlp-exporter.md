---
id: adw-m3-02-otlp-exporter
type: feat
status: done
priority: 2
created: 2026-07-14
epic: adw-m3
depends: [adw-m3-01-tracing-engine]
attempts: []
---
# Filesystem OTLP-JSON span exporter

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`.

## Context

Plan §5 "Traces": `runs/<runId>/spans/` holds OTLP-JSON
`ExportTraceServiceRequest` files, so any standard tracing backend ingests
a run without code changes (S5.3, decision 9). The exporter is the injected
sink of the m3-01 engine; the no-op exporter is the disable mechanism that
keeps tracing a removable middleware (plan §2 watch item). The CLI wiring
step touches `src/cli.ts` — orchestrator-owned integration (M2 pattern:
disjoint file ownership; builders keep to the leaf module).

## Deliverables

- `test/observability/span-exporter.test.ts` (first, red — pinned OTLP-JSON
  fixture/snapshot)
- `src/observability/span-exporter.ts` (filesystem + no-op exporters)
- `src/cli.ts` wiring: tracer + exporter per runId for `adw run` AND the
  ci-round path (orchestrator integration pass)

## Requirements

- [x] `fileSpanExporter(runsRoot, runId)` implements the OTel `SpanExporter`
      contract, writing OTLP-JSON `ExportTraceServiceRequest` files under
      `runs/<runId>/spans/` — hex-encoded trace/span ids, unixNano string
      timestamps, per the OTLP protobuf-JSON mapping; shape pinned by a
      snapshot test over a deterministic span tree (m3-01 id injection)
- [x] Spans are flushed by run end — SimpleSpanProcessor + explicit
      forceFlush (the TLS Lambda-safe pattern) — so a run killed at any
      point keeps every span exported before the kill (S2.6 "artifacts
      flushed"; metric 3 covers blocked/aborted runs too)
- [x] `noopExporter` drops everything; selected by the
      `ADW_DISABLE_TRACING=true` kill-switch (mirrors the TLS pattern) —
      a test proves pipeline outcome + journal are identical either way
- [x] Export failure warns on stderr and never fails the run (Art. VI is
      satisfied by construction when healthy; a sick exporter must not
      convert an otherwise-green run into blocked)
- [x] CLI wiring: both `runSelected` and `ciRound` construct the tracer
      with the filesystem exporter for their own runId — no lane or node
      file changes (nodes stay tracing-unaware, m3-01 grep-guard).
      ciRound receives a late-bound `tracing(ticketId, ciRunId)` factory
      (the mini-lane mints its runId internally); the grep guard holds —
      ci-round.ts types the tracer via the engine's `RunOptions["tracer"]`.
      OTLP-viewer spot-check rides the m3-05 replay audit (Verify note).

## Build protocol (Art. I)

1. Red tests: OTLP-JSON snapshot; flush-on-partial-run; no-op equivalence;
   export-failure tolerance. Validator review per protocol.
2. Implement to green; orchestrator lands the cli.ts wiring after.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green; spot-check one
exported file loads in a standard OTLP viewer (e.g. `otel-desktop-viewer`
or Jaeger OTLP file ingest) — may ride the m3-05 replay audit if tooling is
not at hand.

## Out of scope

TRACEPARENT propagation + span typing (adw-m3-03); capture (adw-m3-04).
