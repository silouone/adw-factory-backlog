---
id: adw-m3
type: epic
status: done
priority: 2
created: 2026-07-14
depends: [adw-m2]
children:
  - adw-m3-01-tracing-engine
  - adw-m3-02-otlp-exporter
  - adw-m3-03-traceparent
  - adw-m3-04-clens-capture
  - adw-m3-05-replay-audit
  - adw-m3-06-review-fixes
attempts: []
---
# EPIC M3 — Observability trio

Complete the three layers: OTel-native tracing with a filesystem OTLP-JSON
exporter, W3C trace-context propagation into agent processes, and cLens
capture injection verified live. The journal (layer 1) exists since M1.

**Children refined 2026-07-15 at pickup** (plan §8, protocol §3) against
the as-built M2 seams — each carries full requirements + build protocol
and awaits operator approval before `in-progress`.

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m3-01-tracing-engine | trace per runId, span per node, no-op default | S5.3, Art. VI |
| adw-m3-02-otlp-exporter | filesystem OTLP-JSON sink + kill-switch, CLI wiring | S5.3, S2.6 |
| adw-m3-03-traceparent | TRACEPARENT via ctx merge, AGENT/TOOL typing | S5.3, S4.3 |
| adw-m3-04-clens-capture | SDK-hook injection, run-scoped sink, live check | S5.2, S5.4 |
| adw-m3-05-replay-audit | reconstruction from artifacts alone + runbook | metrics 3+5 |

## Build plan (refinement notes, 2026-07-15)

- **Ordering:** m3-01 blocks m3-02/03 (engine seam first); m3-04 depends
  only on adw-m2 and runs in parallel from the start; m3-05 is the exit
  gate. m3-03 also depends on m3-02 (its live verify reads exported files).
- **Ownership (M2 pattern — disjoint files):** orchestrator owns m3-01,
  m3-05, and every `engine.ts`/`cli.ts` integration pass; builders take
  the leaf modules (span-exporter.ts, clens.ts). m3-03 and m3-04 both
  touch `build.ts` (`AgentQueryOptions`) — serialize that file or let the
  orchestrator merge.
- **Risky set for operator red-test review (protocol §4):** m3-03
  (cross-process seam M4/M5 inherit) and m3-04 (live-injection seam,
  plan §6 risk). Operator may delegate to validators, as in M2.
- **Amendment in flight:** m3-03 proposes a plan §5 wording change
  (TRACEPARENT rides the engine's per-node ctx merge, not static
  `agentEnv()`) — approve or reject at its refinement gate.
- Live steps (m3-03 verify, m3-04 live check, m3-05 audit run) spend real
  tokens/network: explicit operator go-ahead each time; launch with
  `env -u GITHUB_TOKEN -u GH_TOKEN`.

## Exit criteria

A full run reconstructed from journal + spans + captured sessions alone,
without rerunning anything (m3-05 checklist all green, gaps fixed
in-epic). Milestone exit review: operator `/code-review ultra` + external
review per protocol.
