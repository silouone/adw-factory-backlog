---
id: adw-fe-11-live-bars
type: manual
status: queued
priority: 2
created: 2026-09-12
depends: [adw-fe-08-live-sse, adw-fe-09-node-drawer, adw-fe-10-metrics-rail, adw-fe-12-wire-the-run-screen]
attempts: [{"runId":"adw-fe-11-live-bars-1789367494234","branch":"adw/adw-fe-11-live-bars","workspace":"/Users/silouane/adw-factory/runs/adw-fe-11-live-bars-1789367494234/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/34","provider":"claude","model":"sonnet"}]
---
# The operator-gated live bars for the v1.2 view

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

**Read this before marking anything done.** The repo has a documented failure
mode here: `adw-m5-06` is `done` with its billed live bar still owed and
untracked, recorded only in the ticket's own run log. This ticket exists so the
v1.2 bars cannot go the same way — they are **operator-executed**, not offline
suites, and no offline evidence discharges them.

## Requirements (each is a real dispatched run, discharged by the operator)

- [ ] **Persistence** — a real run writes all four prompts, `system/init`
      reaches the journal, and a `feat` run yields three distinct prompt bodies.
- [ ] **Heartbeat** — a real run emits beats at cadence and stops cleanly at
      every terminal path.
- [ ] **The view** — `adw web` serves grid and Gantt over the banked runs, and
      a live run updates in place without a refresh.
- [ ] **The honest-render bar — the acceptance test for the whole
      amendment.** With a run deliberately **hard-killed mid-build**, the grid
      shows `unknown` **within 60 seconds**. This is the failure the operator
      already hit by hand, and the reason any of this was worth building.

## Verify

- [ ] Each bar above, discharged on a real run and its evidence recorded in
      this ticket's run log — run id, what was observed, and when.
- [ ] Anything still owed when the code lands is **carved into its own
      ticket**, never left implied.

## Run log

### 2026-09-14 — mis-dispatch caught during planning; retyped, not built (run `adw-fe-11-live-bars-1789367494234`)

All supporting code is done and merged: `adw-fe-01` through `adw-fe-10` are
every one `status: done` (persistence, agent config capture, heartbeat, grid,
Gantt, tool dots, live SSE, node drawer, metrics rail). This ticket adds no
code, no route, no projection of its own — its whole body is the four
operator-executed acceptance bars quoted above.

This run was dispatched under the pre-`manual` `type: feat`. That is the exact
mis-dispatch `adw-ticket-kind-operator-executed` (PR #5) was built to
prevent — a `type: manual` ticket with no registered lane, refused
pre-engine by `RESERVED_OPERATOR_TYPES` in `src/intake/ticket.ts`, so the CLI
never spends an agent turn on a ticket only a human can discharge. It was
caught here during planning rather than at gates. **Recommendation to the
operator, not an assertion of certainty about original intent:** retype
`feat` → `manual`, as done in this diff. This is proposed via the normal PR
review path, not applied unilaterally to main.

`depends:` is corrected to add `adw-fe-12-wire-the-run-screen` (created
2026-09-14, still `queued`): "the view" bar requires `adw web` to serve grid
**and** Gantt, which is false today (`src/web/server.ts:66` — one route).
Only that one bar depends on `fe-12`; Persistence, Heartbeat and the
honest-render bar depend only on a real run over the grid, which is already
wired, and are operator-dischargeable now.

No requirement checkbox above is checked — by this ticket's own Context
section, no offline evidence discharges them, and none was produced here.
Nothing is carved into a new ticket: `fe-11` already *is* the tracking ticket
for these four bars, so splitting them out again would be circular.

This file sets `status: blocked` per `tickets/README.md`'s protocol, but that
is the **proposed end-state carried in this PR, not this run's actual
outcome**: gates are green and this diff is clean, so the build harness's own
`open-pr` node will finalize this run as green — committing `status:
in-review` directly to the live repo's main and opening a PR — before this
file's `blocked`/`manual` content ever reaches main. Until the operator
reviews and merges, `adw-fe-11-live-bars` shows `type: feat`, `status:
in-review` with an open PR on main; the retype and the blocked status land
only on merge. (The merge itself will need to reconcile a status-line
conflict — branch says `blocked`, main says `in-review` — a one-line manual
resolution, not a design question.) Only the **view** bar is structurally
blocked (on `fe-12`); the other three are runnable today and just await the
operator actually dispatching and observing them.

**Flag for the operator, not resolved here:** the retype from `feat` to
`manual` happens on the SAME ticket this SAME run (`in-progress` at dispatch)
is attempting to close out. Verified from source, not assumed: this is safe.
`sync-pr-state`'s sweep (`src/pipeline/nodes/sync-pr-state.ts`) parses every
ticket in `tickets/` on each invocation; a `type: manual` ticket fails
`parseTicket` on the `type` field, and `classifySkip` explicitly classifies
that rejection as `"no-lane"` — a documented, non-defect, silently-skipped
case (this repo already carries one live `type: manual` ticket,
`adw-m5-07-remote-ci-round-live-bar`, so the live sweep already exercises this
path today). Separately, this run's own terminal finalizer
(`finalizeTerminal`/`statusAtHead` in `src/cli.ts`) never parses `type` at
all — it only greps the `status:` line off `target.repo`'s HEAD — so it
cannot be affected by this edit either. Watching this run's finalize step is
still worth doing, since it is the first time a ticket has been retyped to
`manual` while its own dispatching run was `in-progress`.

**Test-hardening stage:** confirmed no code was added by this ticket — no
feature exists here to harden, and per the Context section above, no offline
test could discharge an operator-executed bar anyway. No tests added. Gates
run clean on the unmodified diff: `bun run lint`, `bunx tsc --noEmit`,
`bun run test:unit` (547 pass), `bun run test` (1450 pass, 0 fail, ~353s).
