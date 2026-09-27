---
id: adw-par-01-repo-commit-lock
type: feat
status: done
priority: 1
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-par-01-repo-commit-lock-1789164087662","branch":"adw/adw-par-01-repo-commit-lock","workspace":"/Users/silouane/adw-factory/runs/adw-par-01-repo-commit-lock-1789164087662/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Serialize the ticket-file commit edge so two runs can execute in parallel

> Minted 2026-09-11 while scoping the live factory view
> (`ai_docs/2026-09-11-live-factory-view-reflection.md`). The FE's macro grid
> is a fleet monitor only if the fleet can have more than one member; today it
> cannot. This ticket removes the **one** mechanical blocker to concurrent
> runs on *different* tickets. It does NOT build a scheduler.

## Spec amendment — LANDED, pickup is unblocked

`specs/adw-v1.md` §2 listed **"parallel ticket execution"** among the v1
non-goals. It was amended 2026-09-11 (operator-approved) to narrow that to
**"a queue drainer or scheduler"**: the operator stays the scheduler, but two
operator-initiated runs against **different** tickets may execute
concurrently. The amendment text lives in `adw-v1.md` §2 itself — read it
there, not here.

Scope stays deliberately narrow: **no queue, no drainer, no watcher, no
scheduler, no `--parallel` flag.** Two terminals, two tickets.

## Context — what actually blocks parallelism

The per-ticket `O_EXCL` lock (`src/intake/status.ts:116-166`,
`runs/locks/<ticketId>.lock`) solves E7 — *the same ticket* dispatched twice.
It says nothing about two **different** tickets, and nothing else in the
codebase does either.

The real hazard is that ticket-file commits land in the target's **shared
primary working copy** on its default branch. There are **four independent
edges** doing this, each with a byte-similar preamble (branch check → ticket
dirty check → rewrite → pathspec-limited commit):

| # | Edge | Site |
|---|---|---|
| 1 | `transition` (queued→in-progress, →in-review, →done) | `src/intake/status.ts:73-112` |
| 2 | `finalizeBlocked` | `src/intake/attempts.ts:76-120` |
| 3 | `open-pr` — lands **two** commits (attempts, then status) | `src/pipeline/nodes/open-pr.ts:328-440` |
| 4 | `finalizeRejected` (+ reuses `transition` for merged→done) | `src/pipeline/nodes/sync-pr-state.ts:188-230` |

Two concurrent runs racing any pair of these contend on `.git/index.lock`.
The failure is **not** a clean refusal: it is a raw git error surfacing at an
arbitrary moment — including at a terminal transition **at the end of a
90-minute run**, destroying the run's finalize path. The
`adw-m5-06` run log already documents what an unfinalized run costs: a ticket
stranded `in-progress` with `attempts: []`, reset by hand.

Note the second, independent reading of the same table: four copies of one
preamble is an Art. VIII duplication at n=4. The fix for the race and the fix
for the duplication are the same edit.

## Scope — one edge, one lock

**Consolidate the four edges onto a single ticket-commit module, and give that
module a repo-level lock.** Not "add a lock in four places": that would
multiply the duplication it is meant to protect.

## Requirements

**The consolidated edge** (new `src/intake/repo-commit.ts`)

- [ ] One exported function owning the whole window: verify default branch
      checked out (N6) → verify the ticket file is committed-clean (E1) →
      apply a caller-supplied **pure** rewrite → pathspec-limited
      `git commit -- tickets/<id>.md` with a caller-supplied message.
- [ ] The rewrite is a parameter, not a branch: `transition` passes
      `rewriteStatus`; `finalizeBlocked` / `open-pr` / `finalizeRejected` pass
      their own status+attempts rewrite. The module never knows about statuses
      (Art. IX — the pure core stays with each caller).
- [ ] Support a **multi-commit** window: `open-pr` lands two commits and must
      hold the lock across **both**, never re-acquiring between them.
- [ ] Every existing refusal message is preserved verbatim. These are tested
      strings and operator-facing; this ticket is not a rewording.

**The repo lock**

- [ ] `O_EXCL` lockfile at `runs/locks/_repo.lock` — one per target repo,
      distinct from the per-ticket dispatch locks already in that directory.
      The name must not collide with any legal ticket id.
- [ ] **Bounded wait with retry**, not immediate refusal. A commit window is
      milliseconds; a waiter must block and retry, because a run that has
      done 90 minutes of work must not be lost to a sibling's dispatch. Wait
      bound is a named constant with a stated rationale.
- [ ] The lock file records `{runId, pid, acquiredAt}` so a wait timeout
      names its holder rather than reporting an anonymous stall.
- [ ] Released on **every** path — success, refusal, and thrown error.
- [ ] **Lock ordering is documented and enforced:** the per-ticket dispatch
      lock is always the OUTER lock, the repo commit lock always the INNER
      one. Never the reverse — that ordering is the deadlock proof, and it
      belongs in the module header.
- [ ] A wait timeout fails fast with a descriptive error carrying `ticketId`,
      the operation, and the holder's `runId` (Art. IX). The lock is **never
      stolen** — same stance as the existing dispatch lock.

**Reclamation**

- [ ] `adw clean` reclaims a stale `_repo.lock` whose recorded pid is no
      longer alive, reporting what it reclaimed. Without this, one crashed
      run wedges every future commit in the repo.

## Verify

- [ ] **The concurrency test is the point of this ticket.** Against a git
      fixture repo, two concurrent commit windows on **different** tickets
      both complete: two commits land, both ticket files are correct, neither
      call throws. This test must be red before the lock exists.
- [ ] A second concurrency test: a window held past the wait bound produces
      the descriptive timeout error naming the holder's `runId` — not a raw
      git error.
- [ ] Lock released after a **throwing** rewrite (fault injection): the next
      acquire succeeds.
- [ ] `open-pr`'s two commits are proven to occur inside **one** lock
      acquisition.
- [ ] **Unregressed:** the existing suites for all four edges stay green with
      no assertion changes —`test/intake/status.test.ts`,
      `test/intake/attempts.test.ts`, `test/pipeline/nodes/open-pr.test.ts`
      (incl. its snapshot), `test/pipeline/nodes/sync-pr-state.test.ts`.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Self-hosting hazard (read before dispatching this ticket at itself)

This ticket modifies the exact machinery a **self-target run uses to commit its
own status transitions**. The lane process is not at risk — it loads `src/` at
process start and the agent edits a worktree copy — but the **test suite is**:
a test that acquires, holds or removes a lock under the real `runs/locks/`
would contend with its own parent run's dispatch lock, and a test that removes
`_repo.lock` could unblock a window the parent is inside.

- [ ] Every lock test uses an **isolated temp locks directory**, never the
      repo's real `runs/locks/`. No test may read, write or delete a path
      under the live locks dir.

The same caution applies to git: fixture repos only, never the checkout.

## Out of scope

- Any scheduler, queue drainer, watcher, or `--parallel` flag. The operator
  runs two terminals.
- Parallelism of the **same** ticket — still forbidden, still E7, the
  per-ticket lock is unchanged.
- Any FE work. This ticket is a precondition the FE benefits from, not FE
  scaffolding.
- Concurrency hazards outside the commit edge. Run ids (`<ticketId>-<ms>`),
  worktree paths (`runs/<runId>/workspace`) and branch names
  (`adw/<ticketId>`) are already unique per ticket and were checked; if the
  build surfaces another shared-state hazard, **stop and report it** rather
  than widening this ticket.

## Run log

**2026-09-12 — run `adw-par-01-repo-commit-lock-1789164087662` (worktree,
claude, sonnet) reached `test` GREEN and then ABORTED on the turn ceiling:
422 turns against the feat lane's cap of 400** (`feat.ts:116`). 84 minutes.

The `test` node ended `outcome: "next"` — it succeeded. The ceiling tripped at
the node *boundary* on the cumulative count, so the run died **after the work
was complete and before `gates`**: all of the cost, none of the delivery. The
factory finalized correctly (`status: blocked` with a proper `attempts:` entry).

**The work was verified by hand and salvaged, not discarded.** Independent
re-verification in the worktree: `bun run lint` clean (85 files),
`bunx tsc --noEmit` clean, `bun test` **873 pass / 4 skip / 0 fail** across 877
tests. Substance checked against every requirement above: all four edges call
`commitTicketFile` (6/3/8/3 call sites) with **zero raw `git commit` calls
remaining**; lock ordering documented with its deadlock proof; `_repo.lock`
with 5 s bounded wait, 25 ms poll and a holder record; open-pr's two commits
inside one acquisition and never across `gh pr create`; `adw clean`
reclamation present; lock tests use isolated temp dirs, honouring the
self-hosting hazard above.

PR opened by hand: https://github.com/silouone/adw-factory/pull/6 (`b16c9f0`,
18 files, +1909/−239). Status set to `in-review` to reflect reality.

**NOTE:** `sync-pr-state` will SKIP this ticket rather than reconcile it,
because the PR is not recorded in `attempts:` — the entry there records the
blocked factory attempt, which is what actually happened. Close it out by hand
when the PR merges. Same precedent as PR #1 / `adw-m5-06`.

**Factory finding, filed separately:** the last agent stage spent ~70 turns on
hardening tests that all passed against the existing implementation with no
change — the stage that burned the budget returned no defect. See
`adw-caps-01-ceiling-deterministic-tail`.

**2026-09-12 — MERGED.** PR #6 squash-merged as `12c2c78` (18 files,
+1909/−239). Re-verified on merged main: lint clean (85 files), tsc clean.
Status set to `done` BY HAND — `sync-pr-state` skips this ticket because the
PR is not recorded in `attempts:` (that entry records the blocked factory
attempt, which is what actually happened). Same precedent as `adw-m5-06`.
