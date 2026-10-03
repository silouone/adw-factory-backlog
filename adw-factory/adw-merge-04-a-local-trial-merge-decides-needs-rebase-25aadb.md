---
id: adw-merge-04-a-local-trial-merge-decides-needs-rebase-25aadb
type: feat
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: [adw-merge-02-a-rebase-round-holds-the-ticket-lock-26b696]
attempts: []
---
# "Needs rebase" is decided by a local trial merge, and merges are reconciled first

> Spec: `specs/adw-v1.17-mergeable-prs.md` (operator-local, NOT in the repo — this ticket is self-contained; do not look for it), D2 and D4, stories 10–14 and 18.
> The operator chose "rebase only on conflict" on 2026-10-03.

## Context

The sweep triggers a round only when GitHub says `BEHIND` or `DIRTY`. Both
are weak signals:

- `cqc/release-1` has no branch protection, so GitHub never says `BEHIND`.
- Right after a base push GitHub says `UNKNOWN`. PR #174 read `UNKNOWN` on
  two queries hours after it went conflicting.

The kept workspace can answer exactly and immediately: on 2026-10-03,
`git merge-tree --write-tree origin/<base> HEAD` ran in a run workspace in
well under a second (host git is 2.50.1; `--write-tree` needs ≥ 2.38).

## Requirements

- [ ] **R1 — pure decision.** A pure function over `{containsBase, conflicts}`
  returns `current | behind-clean | conflicting`.
- [ ] **R2 — the probe edge.** It fetches the base, runs the ancestry check,
  and only when the branch is behind runs `git merge-tree --write-tree`
  (exit 1 means conflicting; exit 0 means clean; any other exit throws with
  the command and stderr). The working tree is never written.
- [ ] **R3 — trigger.** For a worktree attempt with a kept workspace, a round
  runs on `conflicting`, OR on GitHub `BEHIND`. `behind-clean` returns `open`
  with reason `behind base, no conflicts — mergeable`. For container and
  remote attempts, or a reclaimed workspace, the GitHub `DIRTY`/`BEHIND`
  check stays as it is today.
- [ ] **R4 — failure is visible.** A probe that throws returns `skipped` with
  the reason, never `open`.
- [ ] **R5 — two passes.** Within one target, `syncPrState` processes every
  MERGED/CLOSED ticket before evaluating any OPEN one (each OPEN probe
  re-fetches the base after pass 1). The report order stays deterministic.
- [ ] **R6** — the `UNKNOWN — not a rebase trigger` reason is removed for
  worktree attempts (local truth replaces it).

## Out of scope

- Rebasing behind-clean PRs (out of scope in the spec).
- Local detection for container/e2b attempts.

## Verify

- Pure tests for R1. Sync-seam tests over real git fixtures: current,
  behind-clean (no round), conflicting with GitHub saying CLEAN or UNKNOWN
  (round runs), GitHub BEHIND (round runs), container attempt (GitHub
  fallback), fetch failure (skipped). A two-pass ordering test: a ticket
  merged later in sort order still makes its sibling's probe see the new base.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
