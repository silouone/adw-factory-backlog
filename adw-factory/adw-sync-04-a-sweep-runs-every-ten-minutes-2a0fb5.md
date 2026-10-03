---
id: adw-sync-04-a-sweep-runs-every-ten-minutes-2a0fb5
type: feat
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 90, turns: 200}
depends: [adw-bug-42-a-rebase-round-never-vanishes-mid-session-b782a6, adw-sync-03-adw-sync-can-rebase-without-dispatching-cc7e46]
attempts: []
---
# A launchd job runs the rebase sweep every 10 minutes, 08:00–24:00

> Spec: `specs/adw-v1.17-mergeable-prs.md`, D8, stories 22–27. This
> **narrows the README's "no scheduled runs"**: exactly one scheduled job
> exists, and it never dispatches.

## Context

The operator merges waves one PR at a time, and each merge can make the next
open PR stale. A round takes about 5–15 min (gates, plus maybe one session).
A 10-minute tick keeps a PR shippable about 25 min after a merge in the worst
case, at about 60 cheap `gh pr view` calls an hour across all targets.

## Requirements

- [ ] **R1 — pure renderer.** It renders a LaunchAgent plist: a label,
  `ProgramArguments` running `adw sync --rebase --all-targets` from this
  checkout, `StartInterval` 600, an **explicit PATH** (bun, gh, git, codex,
  docker resolve as in the operator's shell; see the Bun 1.2.4 PATH trap), no
  `GITHUB_TOKEN` in `EnvironmentVariables`, and stdout/stderr appended to
  `~/adw/logs/sync.log`.
- [ ] **R2 — active hours in code.** Pure `isActiveHour(now, {from: 8, to: 24})`.
  An out-of-window tick exits 0 after one log line and makes no gh or git
  calls. The window is a constant, not config.
- [ ] **R3 — operator recipe.** `just schedule-sync install|uninstall|status`
  writes the plist to `~/Library/LaunchAgents/` and runs
  `launchctl bootstrap`/`bootout`/`print`. `status` shows the last exit and
  the log tail.
- [ ] **R4 — no stacking.** launchd never runs two instances of one label;
  adw-merge-02's lock covers overlap with interactive `adw run`s. Document both
  in the recipe's help.
- [ ] **R5** — each tick's log block starts with an ISO timestamp line.

## Out of scope

- Webhooks, tunnels, notifications.
- Auto-installing the job (the operator runs `install`).

## Verify

- Pure tests: the plist snapshot has the interval, PATH, log path and no
  `GITHUB_TOKEN`; `isActiveHour` at 07:59, 08:00, 23:59 and 00:00.
- Manual: `just schedule-sync install`, wait one tick, `just schedule-sync
  status` shows exit 0 and a sweep report in the log. Then `uninstall`.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
