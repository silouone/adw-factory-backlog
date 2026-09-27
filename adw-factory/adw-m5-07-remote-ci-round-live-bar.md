---
id: adw-m5-07-remote-ci-round-live-bar
type: manual
status: queued
priority: 3
created: 2026-09-11
epic: adw-m5
depends: []
attempts: [{"runId":"adw-m5-07-remote-ci-round-live-bar-1789113177246","branch":"adw/adw-m5-07-remote-ci-round-live-bar","workspace":"/Users/silouane/adw-factory/runs/adw-m5-07-remote-ci-round-live-bar-1789113177246/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Live bar: remote CI repair round on a reattached sandbox

`type: manual` (reserved, no registered lane — `adw-ticket-kind-operator-executed`):
this ticket is OPERATOR-EXECUTED, not agent-executed. It is a verification
bar, not work an agent performs: it is discharged by a human launching `just
run-remote <a scratch ticket>` and OBSERVING a repair round happen inside a
reattached E2B sandbox. It is BILLED. The CLI refuses to dispatch it
pre-engine, naming why.

It was dispatched by mistake on 2026-09-11 (`…-1789113177246`, worktree) and
blocked at gates after a build agent worked on an unsatisfiable task — the
prior interim guard was a `DO NOT DISPATCH` banner, since replaced by the
`type: manual` refusal itself.

## Context

Carved out of `adw-m5-03-remote-ci-round` on 2026-09-11 (Tier 0.4). That
ticket is code-complete and offline-green; this is the one thing it owed and
could not discharge locally.

**Operator-gated and BILLED.** E2B is selected declaratively by a human via
`just run-remote` / `--isolation remote`; it is never the default, and the CLI
refuses remote isolation outright without `E2B_API_KEY`.

## The bar

A remote-run scratch ticket with a deliberately CI-red PR completes one
in-sandbox repair round on the RESUMED session:

- [ ] The repair agent runs IN the reattached sandbox (`Sandbox.connect` on
      the paused sandbox, id read from the run's reclaim breadcrumb) — not a
      fresh sandbox, which would break `query({resume})`.
- [ ] The re-push lands via the host-side edge on a freshly minted token.
- [ ] No token is in the sandbox at rest.

## Two known ways this reads as a failure when it is not

Both were predicted in adw-m5-03's own analysis; verify rather than assume:

- **The m4-04 koan.** A resumed agent may read a marker-forced CI failure as
  intentional and take the E8 retrigger path → blocked-after-cap BY DESIGN.
- **The paused-sandbox survival window.** If the PR-open → CI-red → re-run
  delay exceeds E2B's retention, `missing → blocked (no charge)` is the
  CORRECT path, not a bug. Measure the window on the bar.

## Verify

The run's journal shows the repair round executing against the reattached
sandbox id recorded in `workspace.json`, and the push landing.

## Out of scope

`adw-m5-06`'s separate owed bar (`capture ok:true` with the transcript copied
out) — a DIFFERENT live bar on the same lane.
