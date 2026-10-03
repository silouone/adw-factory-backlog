# Amendment v1.17: every factory PR in the review queue is shippable

> **Status:** PROPOSED 2026-10-03 from an operator session, `ready-for-agent`
> once approved. Decisions D1–D9 below were made or confirmed by the operator
> in that session.
> **Amends:**
> - ticket `adw-merge-01` decision **D3** ("a conflict in the lane spawns no
>   agent; the unrebased branch still ships") is **lifted**;
> - ticket `adw-sync-01` ("`adw sync` never runs an agent") is **narrowed**:
>   `adw sync` still never dispatches, but with `--rebase` it may run the
>   rebase maintenance round;
> - the README's "deliberately absent: no scheduled runs" is **narrowed**:
>   exactly one scheduled job exists, a maintenance sweep. It never dispatches.
>
> **Binding:** `constitution.md` is unchanged. The factory still never
> merges (Art. IV). Marking its own PR draft or ready, and labelling it, is PR
> metadata on the factory's own branch, not a merge. No new agent node *type*:
> the resolve session is a build-type node, so Gate III is unchanged.
> **Evidence:** this session's investigation, recomputed from every banked
> journal in `runs/` and live `gh` queries on 2026-10-03.

## Problem Statement

The operator opens a factory PR to review or ship it, and finds GitHub's
"This branch has conflicts that must be resolved". The time spent opening the
PR is wasted. The operator then rebases by hand, which is the work the factory
was supposed to own. This happened on PR #174 (cqc-fe-39), and it happens on
every parallel wave.

The operator believed this was solved. `adw-merge-01` (PR #162) did ship two
halves. Measured, neither closes the gap:

1. **The end-of-run rebase detects the conflict and ships it anyway.** The
   `rebase-onto-base` node re-fetches the base between `commit` and `push`.
   Clean replays work: 31 of 31 kept their PR current silently. But on a
   conflict it aborts, journals `rebase-conflict`, and pushes the unrebased
   branch by design (D3). **6 of 6** lane conflicts ended `green` with a
   conflicting PR: cqc-be-30, cqc-be-35, sabado-38, cqc-fe-45, cqc-fe-37 and
   cqc-fe-39. Every one cost the operator a hand rebase.
2. **The rebase round that should fix it almost never runs.** It is wired into
   exactly one place: the PR-state sweep at the start of `adw run`, on the
   same target. `adw sync` passes no rebase hook, and nothing polls. A
   conflicting PR stays conflicting until the operator happens to dispatch
   another ticket on that target. #174 sat conflicting with no cmc run
   after it.
3. **The one production round died, and it used up the attempt.** cqc-fe-45 on
   2026-10-02 23:14 went through rebase, conflict, charge, then
   `rebase-resolve` (codex). The journal stops 15 s into the resolve session,
   with no `run-end`. The kept workspace was left mid-rebase until the operator
   finished it by hand at 23:47. The cap is charged before the agent runs, so
   this round used up the one allowed resolve. The cqc-fe-44 dispatch started
   27 ms after the last journal line, which points to the round having been
   abandoned in-process. **The root cause is not verified.**
4. **Parallel runs race the round.** The round takes no lock. Three `adw run`s
   on one target each run the sweep, and can each try to rebase the same kept
   workspace.
5. **GitHub's merge state is a weak trigger.** On a base branch without
   "require branches to be up to date" protection, GitHub never reports
   `BEHIND`. `cqc/release-1` has no protection at all. Right after a base push,
   GitHub reports `UNKNOWN` until something asks; #174 read `UNKNOWN` on two
   later queries. `DIRTY` does outrank `BLOCKED`: dependabot #23 on cmc's
   protected `master` reads `DIRTY`. So the trigger misses nothing it can see,
   but it often sees nothing.
6. **The operator cannot see it coming.** The `adw web` PR index queries
   `number,state,headRefName,statusCheckRollup,reviewDecision`, with no merge
   state. A conflicting PR looks the same as a shippable one.

## Solution

From the operator's side: **a factory PR in "ready for review" is mergeable.**
If the factory could not make it mergeable, the PR is a **draft** labelled
`adw:needs-rebase`, so the operator never opens it by mistake. The work is
still on GitHub. The factory flips the PR back to ready on its own once a
rebase goes green.

Four mechanisms, one rule each:

1. **End of run: resolve, don't ship conflicts.** When the end-of-run rebase
   conflicts, the lane runs one fresh resolve session: the same prompt and
   continue logic as the rebase round. Then it runs the gates on the rebased
   commit, then pushes. If the resolve or those gates fail, the lane restores
   the pre-rebase commit and opens the PR as draft with the label.
2. **Local truth decides "needs rebase".** The factory doesn't trust GitHub's
   lazy, protection-dependent merge state. It asks git in the kept workspace:
   fetch the base, check whether the branch already contains it, and if not,
   run a no-write trial merge (`git merge-tree`). A conflict means a rebase is
   needed. "Behind but clean" means no rebase: that PR is mergeable and only
   shows a "behind" chip. GitHub's `BEHIND` still triggers a rebase, because
   then GitHub blocks the merge. Container and remote attempts have no kept
   local checkout, so they fall back to GitHub's `DIRTY` / `BEHIND`.
3. **A sweep the operator never has to remember.** `adw sync --rebase
   [--all-targets]` runs the maintenance sweep without dispatching anything.
   A launchd job runs it every 10 minutes between 08:00 and 24:00. Each sweep
   handles merges first, then open PRs, so the PRs that a merge made stale are
   re-checked in the same pass that saw the merge. Local detection doesn't
   wait for GitHub to recompute.
4. **Rounds are exclusive and crash-safe.** A round holds the ticket's lock. A
   live holder means skip. A dead holder's lock is reclaimed and journaled. A
   kept workspace found mid-rebase with no live owner is reconciled: abort the
   rebase, journal it, and mark the PR draft with the label. An exhausted or
   failed round no longer moves the ticket to `blocked`. It stays `in-review`,
   so the sweep still advances it to `done` when the operator merges it by hand.

## User Stories

### End of run

1. As the operator, I want a run whose end-of-run rebase conflicts to try one
   resolve session before pushing, so that a parallel wave doesn't hand me
   conflicting PRs.
2. As the operator, I want the gates re-run on the resolved, rebased commit
   before anything is pushed, so that a resolution can't ship red code.
3. As the operator, I want a run whose resolve fails, or whose post-resolve
   gates are red, to open its PR as a draft labelled `adw:needs-rebase`, so
   that I never open a PR I can't ship and the work isn't lost.
4. As the operator, I want that draft PR's body to say why it is a draft (the
   conflicted paths, and resolve failed vs. gates red), so that I know what is
   left without reading the journal.
5. As the operator, I want a clean end-of-run rebase to behave exactly as it
   does today: replay, gates, push, ready PR, so that the 31-of-31 path is not
   disturbed.
6. As the operator, I want the lane's resolve session journaled like every
   other agent node (node-start/end, prompt, usage, capture), so that "why is
   this PR a draft" is answerable from the journal.
7. As the operator, I want the run's terminal outcome to stay `green` when it
   opens a draft PR, with the draft and its reason journaled, so that green
   still means "a PR exists and its tests passed on the branch it holds".
8. As the operator, I want the lane's resolve budget (one session) to be
   separate from the maintenance round's budget (one session per attempt),
   so that a lane failure doesn't leave the round with nothing to try.
9. As a factory maintainer, I want the lane to reuse the rebase round's
   resolve prompt and continue/marker checks rather than a second copy, so
   that the two paths cannot drift (Art. VIII).

### Detection

10. As the operator, I want "needs rebase" decided by a local trial merge in
    the kept workspace, so that a branch with no protection and a lazy
    `UNKNOWN` from GitHub can't hide a conflict.
11. As the operator, I want a PR that is behind but has no conflicts left
    alone, so that every merge doesn't trigger N-1 gate reruns on a
    docker-heavy target.
12. As the operator, I want GitHub's `BEHIND` to still trigger a rebase, so
    that a repo requiring up-to-date branches doesn't leave the PR
    unmergeable.
13. As the operator, I want container and remote attempts to fall back to
    GitHub's `DIRTY`/`BEHIND`, so that they are not silently excluded.
14. As the operator, I want a detection that can't run (fetch fails, workspace
    reclaimed) reported as a skipped outcome with its reason, so that silence
    never means "fine".

### The sweep

15. As the operator, I want `adw sync --rebase` to run the rebase round on
    every in-review ticket whose PR needs one, so that I can make the queue
    shippable without dispatching a ticket.
16. As the operator, I want `adw sync --rebase --all-targets` to cover every
    configured target in one invocation, so that one command fixes the whole
    queue.
17. As the operator, I want `adw sync` without `--rebase` to behave exactly as
    today, with no agent and no push, so that a reconcile-only sweep stays
    zero-cost.
18. As the operator, I want each sweep to reconcile MERGED/CLOSED PRs before
    it evaluates OPEN ones, so that the PRs a merge just made stale are
    handled in the same pass.
19. As the operator, I want `--dry-run` and `--rebase` refused together with a
    clear message, so that a preview never spends an agent session.
20. As the operator, I want the sweep's report to show one line per ticket:
    rebased, needs-human, skipped (with reason) or open, so that I can read the
    queue's state at a glance.
21. As the operator, I want a codex attempt whose sweep runs without Codex
    auth to be skipped without charging a round, so that a missing login
    doesn't use up the budget (existing behaviour, kept).

### Schedule

22. As the operator, I want a launchd job that runs `adw sync --rebase
    --all-targets` every 10 minutes, so that stale PRs are fixed without me
    remembering.
23. As the operator, I want the job to do nothing outside 08:00–24:00, with
    that window checked in code, so that nothing runs while nobody merges.
24. As the operator, I want one recipe to install, uninstall and show the
    status of the job, so that turning it on or off is one command.
25. As the operator, I want the job's output appended to a log under
    `~/adw/logs/`, so that I can see what the last sweeps did.
26. As the operator, I want the job to run with an explicit PATH and without
    `GITHUB_TOKEN`, so that `gh`, `git`, `bun` and `codex` resolve as they do in
    my shell.
27. As the operator, I want a tick that starts while the previous one is still
    running to be harmless, so that a slow gates run can't stack sweeps.

### Locking and crash safety

28. As the operator, I want a rebase round to hold the ticket's lock for its
    whole duration, so that two parallel `adw run`s or a scheduled sweep never
    rebase the same workspace at once.
29. As the operator, I want a sweep that finds the lock held by a live process
    to skip that ticket with "locked by <runId>", so that it's visible and
    retried on the next tick.
30. As the operator, I want a lock whose owning process is dead to be
    reclaimed and the reclaim journaled, so that a crash doesn't freeze the
    ticket forever.
31. As the operator, I want a kept workspace found mid-rebase with no live
    owner to have its rebase aborted and the PR marked draft with the label,
    so that a crashed round leaves a clean, honest state.
32. As the operator, I want a crashed round's journal closed with a
    synthesized end event, so that no rebase run appears in-flight forever.
33. As the operator, I want the crash-safe charge kept (charged before the
    agent runs), so that a crash loop can't spend unbounded sessions.
34. As the operator, I want the cause of the cqc-fe-45 round's silent death
    found and covered by a failing-first test, so that the round isn't
    abandoned mid-session again.

### Ticket and PR lifecycle

35. As the operator, I want a round that fails, or that finds its budget
    spent, to leave the ticket `in-review` and mark the PR draft with the
    label, so that my hand-merge still moves the ticket to `done`.
36. As the operator, I want a green round on a draft labelled PR to mark it
    ready and remove the label, so that it re-enters my queue only when
    shippable.
37. As the operator, I want the factory to touch draft/ready state and the
    label only on PRs whose head branch it owns, so that a PR I drafted by hand
    is never flipped.
38. As the operator, I want every draft/ready/label change journaled with its
    reason, so that the PR's history matches the run's.
39. As the operator, I want a hand-pushed branch, where the remote head is not
    the kept workspace's HEAD, still refused by the lease, so that the factory
    never overwrites my manual fix (existing D4, kept).

### Visibility

40. As the operator, I want the `adw web` PR chips to show "conflicts" for a
    conflicting PR, so that I can see it before I click.
41. As the operator, I want a "behind" chip for a behind-but-mergeable PR, so
    that I know the gates ran on an older base.
42. As the operator, I want a "draft · needs rebase" chip for a PR the factory
    parked, so that the queue separates shippable from parked.
43. As the operator, I want an `UNKNOWN` merge state to render no chip rather
    than a guess, so that the board never lies.

## Implementation Decisions

- **D1 — End-of-run resolve (operator, 2026-10-03).** On a lane conflict, the
  lane runs one fresh resolve session in place, then continue, then the gates
  on the rebased commit, then push. If the resolve fails, a marker remains, a
  later commit re-conflicts, or the post-resolve gates are red, the lane resets
  the workspace to the pre-rebase commit and pushes that. Then `open-pr` opens
  a **draft** with the `adw:needs-rebase` label and a body section naming the
  conflicted paths and the failure. The run stays `green`. A clean-rebase
  gates failure is untouched, still a hard stop. The lane reuses the round's
  resolve prompt builder and continue node. The lane resolve has its own budget
  of 1, recorded on the attempt separately from the round's `rebaseRounds`.
- **D2 — Detection is local-first (operator: "only on conflict").** A new pure
  decision over three git facts: contains-base, behind, trial-merge conflict.
  It returns `current | behind-clean | conflicting`. A rebase runs on
  `conflicting`, or on GitHub `BEHIND`. `behind-clean` never rebases. The
  trial merge is `git merge-tree --write-tree` (git ≥ 2.38; the host has
  2.50.1) and writes nothing to the working tree. Container and remote
  attempts use GitHub's `DIRTY`/`BEHIND`, as today.
- **D3 — `adw sync --rebase`.** It adds the rebase hook to `runSync` and
  `runSyncAllTargets`, and never the CI hook. With `--dry-run` it is refused.
  The `SyncDeps` interface gains the round's rehydration deps, which `adw run`
  already builds. The builder is factored once and shared by both callers.
- **D4 — Two-pass sweep.** Within one target, `syncPrState` reconciles every
  MERGED/CLOSED ticket first, then evaluates OPEN ones. The sweep re-fetches
  each OPEN PR's base after pass 1. That replaces the earlier "re-check
  `UNKNOWN` after a minute" idea: local detection doesn't depend on GitHub
  having recomputed.
- **D5 — Ticket lock for maintenance.** A round acquires the same per-ticket
  `O_EXCL` lock file that dispatch uses, in the same `locksDir`. The lock file
  carries `{pid, runId, startedAt}`. A held lock with a live pid means skip. A
  dead pid means reclaim, journaled. The ticket author must verify that the
  dispatch lock is released once the ticket is `in-review`, and record it.
- **D6 — Crash reconciliation.** Before probing, the round checks the kept
  workspace for a rebase in progress. If found, and the lock was dead or
  absent, it aborts the rebase, appends a synthesized end to the orphaned
  rebase run's journal (the `synthesized-exit` precedent), and parks the PR
  (D7). The charge is **not** refunded.
- **D7 — Park, don't block.** A round that fails, or finds its budget spent,
  marks the PR draft, adds `adw:needs-rebase`, and returns `needs-human`. The
  ticket stays `in-review`. This replaces today's `finalizeBlocked` on
  round failure, which stopped the sweep from ever seeing the operator's merge.
  A green round on a parked PR runs `gh pr ready` and removes the label. All
  three gh writes are guarded by "head branch equals the attempt's branch".
- **D8 — Schedule.** A pure renderer produces the LaunchAgent plist:
  `StartInterval` 600, explicit PATH, no `GITHUB_TOKEN`, and stdout/stderr
  appended to `~/adw/logs/sync.log`. A pure `isActiveHour(now, {from: 8, to:
  24})` makes an out-of-window tick exit 0 with one log line. `just
  schedule-sync install|uninstall|status` wraps `launchctl`. launchd never
  overlaps instances of one label, and D5 covers any other overlap.
- **D9 — Web chips.** The PR index's `gh pr list` query adds
  `mergeStateStatus,mergeable,isDraft,labels`. A new pure chip projection maps
  them: `conflicts` for DIRTY or CONFLICTING, `behind` for BEHIND, `parked`
  for draft + label, and nothing for UNKNOWN. The web stays read-only.
- **Root cause of the fe-45 death (story 34).** This is a separate bug ticket
  and must be fixed before the schedule ships. Reproduce with a failing test
  first. Hypotheses to rule in or out: the codex resolve path resolving
  without a node-end; an unawaited promise in the sweep; an external kill.

## Testing Decisions

- **A good test asserts what the operator or GitHub sees:** journal events,
  ticket-file fields, the gh argv issued, git refs and workspace state. It
  never asserts which helper ran. Real git in temp repos, fake `gh`, fake
  agent query: the existing pattern.
- **Seam 1 — the lane (`runLane` with the real lane spec).** End-of-run
  resolve: clean → unchanged; conflict + good resolve → rebased, gated,
  pushed, ready; conflict + bad resolve → reset, pushed, draft + label + body
  section; gates red after resolve → same. Prior art: the existing rebase node
  tests and the lane tests for chore, bug and feat.
- **Seam 2 — the sync CLI (`runSync` / `runSyncAllTargets` → `syncPrState` →
  `rebaseRound`).** Covers local detection (current, behind-clean, conflicting,
  GitHub BEHIND, and container fallback), two-pass ordering, lock held by a
  live pid or a dead pid, crash reconciliation, park/unpark gh calls, the
  head-branch guard, `--rebase` + `--dry-run` refused, and plain `adw sync`
  unchanged. Prior art: the existing sync-pr-state, rebase-round and CLI sync
  tests.
- **Seam 3 — the web chip projection (pure).** Every
  `mergeStateStatus × mergeable × isDraft × label` combination maps to the
  chip table, and UNKNOWN maps to no chip. Prior art: the existing PR-chips
  tests and `validationChipOf`.
- **Seam 4 — the schedule (pure).** The plist renders with the interval, PATH,
  log path and no `GITHUB_TOKEN`. `isActiveHour` holds at the 07:59, 08:00,
  23:59 and 00:00 boundaries. Installing the job is operator-run, not
  unit-tested.
- **Art. I:** every ticket lands its red tests first. The fe-45 bug ticket
  starts from a reproduction.

## Out of Scope

- Auto-merge, merge queues, any `gh pr merge` (Art. IV stands).
- Rebasing behind-but-clean PRs, including as a per-target option. Revisit
  only with evidence of a semantic break shipped from a stale base.
- GitHub webhooks or a tunnel. The 10-minute poll plus the two-pass sweep is
  the event model.
- Ordering a wave to avoid conflicts (adw-parallel-01 territory).
- Container/e2b local detection (they keep the GitHub fallback).
- Notifications (macOS, Slack) when a PR is parked.
- Refunding a crashed round's charge.

## Further Notes

- PR #174 itself was resolved by hand at 11:08 and merged as `aaab656`. No
  factory PR was conflicting at the time of writing. sabado #1328 is CLEAN.
- The bug ticket for the fe-45 death and the lock ticket must land before the
  schedule ticket: polling multiplies whatever race exists.
- Suggested order: fe-45 root cause, then the lock and crash reconciliation,
  then park/unpark, then local detection with the two-pass sweep, then
  end-of-run resolve, then `adw sync --rebase`, then the schedule, then the
  web chips. End-of-run resolve can run in parallel after park/unpark lands.
