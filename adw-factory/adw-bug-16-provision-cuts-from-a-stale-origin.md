---
id: adw-bug-16-provision-cuts-from-a-stale-origin
type: bug
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 90, turns: 400}
depends: []
attempts: [{"runId":"adw-bug-16-provision-cuts-from-a-stale-origin-1789850372278","branch":"adw/adw-bug-16-provision-cuts-from-a-stale-origin","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-16-provision-cuts-from-a-stale-origin-1789850372278/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/92","provider":"claude","model":"sonnet"}]
---
# Provision cuts the attempt branch from origin/<base> without fetching, so a run builds on whatever the last fetch left behind

> Observed 2026-09-19 on `sabado-01-…-1789842707826`: `sabado-00` (#1216) had
> merged 40 minutes earlier, `sabado-01` declared `depends: [sabado-00]`, the
> depends guard passed (the ledger said `done`), and the branch was still cut
> from a `origin/main` that predated #1216. The agent re-implemented the
> slot machinery on the old script; the PR needed a manual rebase with
> conflicts on every file. `review: false`: hard-gated.

## The defect

`src/workspace/worktree.ts` `resolveBranchPoint` picks `origin/<base>` when
a remote exists, then `git worktree add … origin/<base>` — no `git fetch`
anywhere in provisioning (`grep -n fetch src/workspace/worktree.ts` → none).
The ref is only as fresh as the operator's last manual fetch. A
`depends:`-satisfied ticket therefore has no guarantee its dependency's
code is in its base, which defeats the point of the guard.

## Requirements

- [ ] **R1** Before cutting the branch, provisioning runs
      `git fetch --prune origin <base>` in the target repo, through the same
      `capture` seam every other git call uses. A fetch failure is a
      `WorkspaceError` naming the ticket and `provision` (E5) — never a
      silent fall-back to the stale ref.
- [ ] **R2** The journal's `provision` `node-end` details carry the
      resolved base sha (`baseSha`), so a PR can be traced to the exact
      `main` it was built on.
- [ ] **R3** No fetch when the target has no remote (the test-fixture case,
      `resolveBranchPoint(false, …)`), byte-identical behaviour there.

## Verify

- [ ] Red test: with a fixture remote whose `main` has moved past the local
      `origin/main` ref, provisioning cuts the branch from the remote's new
      head, not the stale ref. RED today.
- [ ] Red test: a fetch that fails (unreachable remote URL) raises a
      `WorkspaceError` whose message names `provision` and the ticket id.
- [ ] Red test: the `provision` node-end details include `baseSha` equal to
      `git rev-parse origin/<base>` after the fetch.
- [ ] Existing worktree suites green unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
