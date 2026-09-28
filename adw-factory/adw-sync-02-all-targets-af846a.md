---
id: adw-sync-02-all-targets-af846a
type: feat
status: queued
priority: 2
created: 2026-09-28
depends: []
attempts: []
---
# Reconciling every target still takes one invocation per target

> Minted 2026-09-28, after the operator asked "when I merge a PR opened by a
> lane, it does not automatically update the ticket? we don't have a watcher
> mechanism plugged to that?"

## Evidence

The answer is no, and `adw-sync-01` is only half the fix. That ticket made
reconciliation reachable without dispatching — `adw sync` / `just sync`
(`src/sync.ts`, `src/cli.ts:1834`) — but it is scoped to **one** target:

```
adw sync --target <name>        # loadTarget(targetsDir, targetName) — exactly one
```

So the two ways a merged PR closes its ticket today are (a) start a run
against that target — `syncPrState` at `src/cli.ts:352`, before selection —
or (b) hand-run `just sync` against that target. Both are pull-based and both
are per-target. Verified on 2026-09-28: no webhook receiver, no poller
(`grep -rl "webhook\|setInterval" src` finds nothing on this path), no
`launchctl list | grep adw`, no crontab entry. Merge a PR, walk away, and the
ticket sits `in-review` until someone remembers which target it belonged to.

That matters because `depends:` is enforced (`adw-depends-enforce`): a ticket
left `in-review` **blocks** its dependents on that target, and `just next`
hides runnable work behind an edge that is already satisfied — the same damage
`adw-sync-01` measured, now multiplied by the number of live targets.

**Measured target set, 2026-09-28** (resolved `ticketsDir`, existence checked):

| target | tickets dir | tickets | in-review |
|---|---|---|---|
| adw-factory | `~/adw/backlog/adw-factory` | 223 | 2 |
| clens | `~/adw/backlog/clens` | 14 | 2 |
| cmc | `~/adw/backlog/cmc` | 13 | 0 |
| content-quality-checker | `~/adw/backlog/cqc` | 13 | 0 |
| sabado | `~/adw/backlog/sabado` | 22 | 0 |
| api-content, bricklane-persona-hooks, claude-home, coorpacademy, coorpacademy-lambda, coorpacademy-oplog, serverless-plugins, translated-language-service | `<repo>/tickets` | — | **dir does not exist** |

**5 of 13** target configs resolve to a ticket store that exists. The other 8
fall back to `<repo>/tickets` and have none. A naive loop over `targets/*.json`
would turn eight absent directories — and any unreachable checkout or
un-authenticated repo among them — into a sweep-wide failure.

Two facts checked so they do not become surprises during the build:

- **The branch guard is already store-aware.** `commitTicketFile` refuses off
  the default branch (N6), but `syncPrState` resolves the label through
  `resolveDefaultStore` → `store.branch` (`sync-pr-state.ts:292-298`,
  `loader.ts:541-548`), not `target.base`. So `cmc` (`base: cqc/release-1`,
  store on `main`) is not a blocker and needs no special case.
- **All five live stores share one git root** (`~/adw/backlog`), and the repo
  lock is one file at `<runsRoot>/locks/_repo.lock`. Concurrency across
  targets would buy nothing but contention.

## Requirements

- [ ] `adw sync --all-targets` reconciles every **eligible** target in one
      invocation. Eligible = the target config parses AND its resolved
      `ticketsDir` exists. An ineligible target is reported as one skip line
      with its reason and does not fail the sweep — the 8 of 13 above must not
      turn a green sweep red.
- [ ] `--all-targets` and `--target <name>` are **mutually exclusive**. Both
      passed → refuse with a descriptive error naming both flags, exit
      `EXIT_REFUSED` (2), no gh call, no write.
- [ ] Targets are swept **sequentially, in a deterministic order** (name,
      ascending). Never in parallel: one shared `_repo.lock`, one shared store
      git root.
- [ ] One target's failure does not abort the rest. A thrown refusal
      (wrong branch, dirty ticket file, lock timeout) or a gh error is caught
      **per target**, reported against that target's name, and the sweep
      continues. This is a deliberate narrowing of `adw-sync-01`'s
      "propagate, don't catch" contract, and applies **only** to the
      `--all-targets` path — `adw sync --target X` keeps propagating exactly
      as it does today.
- [ ] Exit code: `EXIT_GREEN` (0) when every eligible target completed —
      skips alone never make it non-zero; `EXIT_BLOCKED` (1) when at least one
      eligible target failed. A launchd/cron job reads this, so it is part of
      the contract, not an implementation detail.
- [ ] `--dry-run` composes with `--all-targets` and stays total: no commit on
      any target, MERGED/CLOSED outcomes still carry `dryRun: true`.
- [ ] The report stays the one `makeSyncReport`/`syncLine` already render
      (Art. VI), with each target's block introduced by a line naming the
      target. This ticket changes WHICH stores the sweep visits, not what it
      reports per ticket.
- [ ] `just sync-all` (and `just sync-all-dry`) expose it, alongside the
      existing per-target recipes.

## Verify

- `adw sync --all-targets --dry-run` visits the 5 eligible targets, names the
  8 skipped ones with a reason, mutates nothing, exits 0.
- A ticket whose PR merged reaches `done` without naming its target and
  without any dispatch.
- `--all-targets --target adw-factory` refuses, exit 2, nothing written.
- A target whose gh edge throws is reported and the remaining targets are
  still swept; the invocation exits 1.
- Running it twice is a no-op the second time.
- Running it while a dispatch is in flight neither corrupts the ledger nor
  deadlocks against `_repo.lock`.
- `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

- **The scheduler.** This ticket makes one invocation sufficient; wiring a
  launchd job that calls it every N minutes is a separate operator-executed
  ticket, deliberately not bundled here.
- **What `syncPrState` decides** (MERGED → done, CLOSED → rejected, OPEN → no
  write) and its CI-maintenance hook. `adw sync` remains dispatch-incapable.
- **Tickets with no recorded PR.** `recordedPr` reads `attempts[].pr`, so a
  ticket whose PR was opened outside a factory run is invisible to the sweep —
  measured: all 4 tickets currently `in-review` (adw-backlog-03, adw-bug-28,
  clens-007, clens-008) carry no `pr`, and `--all-targets` will not close them.
  That gap is real and belongs in its own ticket.
