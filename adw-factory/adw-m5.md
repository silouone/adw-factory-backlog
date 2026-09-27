---
id: adw-m5
type: epic
status: in-progress
priority: 3
created: 2026-07-14
depends: [adw-m4]
children:
  - adw-m5-01-e2b-template
  - adw-m5-02-e2b-workspace
  - adw-m5-03-remote-ci-round
  - adw-m5-04-remote-run-robustness
  - adw-m5-05-settlement-bound
  - adw-m5-06-remote-capture-parity
  - adw-m5-07-remote-ci-round-live-bar
attempts: []
---
# EPIC M5 — Remote workspace

Third `Workspace` kind: remote sandbox on E2B, behind the same interface
(S4.5, decision 5: E2B beat Daytona/Modal/Blaxel on TS SDK maturity +
key-in-hand). **Post-v1 expansion** (plan §8 as reordered 2026-07-14).
Children are coarse — **refine each at pickup**. m5-01/-02 refined
2026-07-18 (operator-approved: E2B_API_KEY env-only, §5 amendment #6
spike-gated, keep=pause, clean wiring in m5-02); adw-m5-03 minted then
for the deferred remote CI round (the m4-04 precedent).

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m5-01-e2b-template | E2B custom template (bun+git+claude) + spawn spike | S4.5 |
| adw-m5-02-e2b-workspace | `e2b.ts` + clean wiring + reused contract tests | S4.3 S4.5 |
| adw-m5-03-remote-ci-round | CI round for remote attempts (coarse) | S2.4 S4.3 |
| adw-m5-04-remote-run-robustness | Un-wedge teardown + liveness watchdog + sandbox sizing (from the stranded clens-006 live bar) | Art. V S2.5 |
| adw-m5-05-settlement-bound | Settlement bound on teardown | Art. V |
| adw-m5-06-remote-capture-parity | cLens capture parity for the remote lane (async fetchTranscript seam; both lanes) | S5.3 |
| adw-m5-07-remote-ci-round-live-bar | The billed live bar carved out of m5-03 | S2.4 |

## Exit criteria

**This epic cannot reach `done` until `adw-m5-07`'s BILLED live bar runs.**
That is intentional, not an oversight: `tickets/README.md` says an epic is
done when every child is done, and m5 now carries a deliberately deferred,
operator-gated child. A future reader should find that stated rather than
infer it from a stalled epic.

A toy chore completes the full lane under `--isolation remote` with
identical pipeline behavior; remote agent spans parent correctly under the
run (S5.3).
