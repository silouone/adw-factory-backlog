---
id: adw-merge-01-a-pr-that-falls-behind-its-base-is-rebased-by-the-factory-d86902
type: feat
status: blocked
priority: 1
created: 2026-09-29
caps: {minutes: 180, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-merge-01-a-pr-that-falls-behind-its-base-is-rebased-by-the-factory-d86902-1790643026443","branch":"adw/adw-merge-01-a-pr-that-falls-behind-its-base-is-rebased-by-the-factory-d86902","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-merge-01-a-pr-that-falls-behind-its-base-is-rebased-by-the-factory-d86902-1790643026443/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A PR that falls behind its base is rebased by the factory, not the operator

> **Evidence, 2026-09-29, cqc target.** `cqc-be-07` built in parallel with
> `cqc-be-04` and `cqc-be-05`. All three were cut from the same `main`; #17 and
> #18 merged first. When the operator reached #19 it was `CONFLICTING`: four
> incoming commits, two files touched on both sides
> (`backend/src/common/errors.ts`, `errors.test.ts`), one real conflict — both
> sides had **appended** a `describe` block to the same test file and git
> interleaved them. The fix was mechanical (keep both blocks, sort the imports)
> and took a builder session ~10 minutes including the four gates. The operator
> has done this by hand for several PRs in the 2026-09-28 wave; each merge
> makes the next open PR stale, so the same PR can need it twice.
>
> Root cause is structural, not a bug: the factory builds in parallel
> (adw-parallel-01) but the human merges serially (Story 3). Every wave of N
> parallel PRs on one target produces up to N-1 stale PRs. A shared file such as
> `errors.ts` that every ticket appends to guarantees the conflict.

## What the factory does today

- `provision` cuts the attempt branch from a **fresh** `origin/<base>`
  (`src/workspace/worktree.ts`, adw-bug-16). Nothing after that ever looks at
  the base again.
- `push` is `git push -u origin <branch>` (`src/pipeline/nodes/push.ts`). No
  force variant exists.
- `sync-pr-state` queries `gh pr view --json state,comments,reviews` and, for an
  OPEN PR, hands off to the CI hook (`ci-round`), which only looks at
  `gh pr checks`. Neither reads `mergeStateStatus`. A `CONFLICTING` or `BEHIND`
  PR is indistinguishable from a healthy one.
- The ticket reaches `in-review` when the PR opens (S1) and `done` when the
  operator merges (S3). The factory's definition of "finished" therefore stops
  at "a PR exists", not "a PR the operator can merge".

## What to build

The factory owns keeping its own branch mergeable, the same way `ci-round`
owns keeping its own CI green: **maintenance of the existing attempt, never a
dispatch, never a merge.** Two places.

### At the end of the build lane — never open a stale PR

Between the last commit and `push`, re-fetch `origin/<base>` and rebase the
attempt branch onto it if it moved. Gates run **on the rebased commit**; push
follows. A run that took two hours on a busy target must not hand the operator
a PR that is already behind.

### While the PR is open — a rebase round

`sync-pr-state` adds `mergeStateStatus` to its `gh pr view` query. For an OPEN
PR whose status is `BEHIND` or `DIRTY` (GitHub's word for conflicting), the OPEN
branch runs a **rebase round** before the CI check:

```
rebase (fetch origin <base>; git rebase origin/<base> in the kept workspace)
  │ clean ──────────────────────────────► gates ► push --force-with-lease
  └ conflicts ► rebase-resolve (agent, one session, FRESH — perf-06 R1/R2:
                 fed the conflicted hunks + plan.md + the ticket body)
                 ► gates ► push --force-with-lease
```

Through the existing engine, like `ci-round`'s mini-lane: every step journaled
(Art. VI), gates/push one representation (Art. VIII), `maxRounds 0`. A rebase
round and a CI round in the same invocation run **rebase first**: a
force-push re-triggers CI anyway, so checking CI on the stale head is wasted.

## Requirements

- [ ] **R1 — no stale PR opens.** The build lane re-fetches `origin/<base>` after
  its last commit. If `git merge-base --is-ancestor origin/<base> HEAD` is
  false, the branch is rebased and gates re-run on the result before `push`.
  Red test (lane suite): a fixture where `origin/<base>` gains a commit during
  `build` yields a pushed HEAD whose parent chain contains that commit, and a
  `rebase` journal event carrying `{ fromBase, toBase, before, after }` shas.
  A base that did not move yields no rebase and no event.
- [ ] **R2 — sync sees mergeability.** `parsePrView` accepts and exposes
  `mergeStateStatus` (`BEHIND | BLOCKED | CLEAN | DIRTY | DRAFT | HAS_HOOKS |
  UNKNOWN | UNSTABLE`, the GraphQL enum). `UNKNOWN` (GitHub still computing,
  observed for ~30 s after every push) is **not** a trigger and is journaled as
  such. Red test: the fixture stdout with `"mergeStateStatus":"DIRTY"` parses;
  a value outside the enum fails fast naming the ticket (Art. IX).
- [ ] **R3 — the rebase round.** For OPEN + (`BEHIND` | `DIRTY`) the hook runs
  the mini-lane above in the kept workspace. A **clean** rebase costs no agent
  session and is **not capped** (a busy target may need several; each is
  journaled). Conflicts invoke `rebase-resolve` once; that session is
  **charged first** to the attempts entry (`rebaseRounds`, persisted before
  the agent runs, as `ciRounds` is) and capped at **1** per attempt. A second
  conflicted rebase, or a red gate after resolution, ends in `finalizeBlocked`
  with a reason listing the conflicted paths — the operator reads *which*
  files fought, not "rebase failed". Red tests: clean-rebase fixture → gates →
  push, no agent call, ticket stays `in-review`; conflicted fixture → one
  `rebase-resolve` node run, `rebaseRounds: 1` committed before it; a second
  conflicted round → `blocked`, reason contains the path.
- [ ] **R4 — force only with a lease, only on the attempt branch.** A new
  push mode `--force-with-lease=<branch>:<expected-remote-sha>` where the
  expected sha is what the round fetched at its start. The S2.7 gates guard,
  dirty-tree refusal and base/detached-HEAD refusal are **unchanged**. A lease
  failure (someone pushed to the attempt branch meanwhile — the operator
  fixing it by hand) is **not retried**: the round ends `open` with a journaled
  `lease-refused` reason and the local rebase is left in place for autopsy.
  Red tests: the push argv contains the lease; a lease-refused stderr fixture
  yields no retry and action `open`.
- [ ] **R5 — precedence.** MERGED/CLOSED outrank maintenance (unchanged).
  Within OPEN: rebase round, then the CI check on the **new** head, in the same
  invocation only if the rebase pushed; otherwise the CI check runs as today.
  A `DIRTY` PR whose CI is also failing gets **one** rebase round, not a CI
  round — CI on a conflicting head is not evidence. Red test: OPEN + DIRTY +
  failing checks → exactly one round, the rebase one, `ciRounds` untouched.
- [ ] **R6 — the ticket file records it.** The attempts entry gains
  `rebaseRounds` (count of **agentic** resolutions) and `rebased:
  <toBase-sha>` (last base the branch was rebased onto, updated on every
  clean or resolved rebase). `parseTicket` accepts both, absent → 0 / none
  (S6.3 compat with every existing attempt). The board's card (adw-pr-03)
  may read `rebased` later; out of scope here.
- [ ] **R7 — spec amendment, before code.** Story 3 (`specs/adw-v1.md`) says
  the factory never merges, approves or closes. Rebasing and force-pushing
  **its own** attempt branch is none of those, but it is a new write to the
  remote after `in-review`, so it must be written down: add an acceptance
  bullet to Story 3 ("shall keep its open PR rebased onto its base; the
  operator merges what is already mergeable") and a §6 risk row in
  `adw-v1-plan.md` for the lease-refused case. Operator approves the
  amendment before R1's tests are written.

## Out of scope

- Merge queues, auto-merge, `gh pr merge` of any kind — Story 3 stands.
- Rebasing an epic ref (`adw-v1.12-epic-drain.md` leaves that undecided).
- Squashing or rewriting the attempt's commit history beyond what `git rebase`
  does to replay it.
- Ordering the wave to avoid conflicts (adw-parallel-01 territory).

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`, green.
- `grep -n "mergeStateStatus" src/pipeline/nodes/sync-pr-state.ts` ≥ 1.
- `grep -n "force-with-lease" src/pipeline/nodes/push.ts` ≥ 1.
- Live: open two tickets on `targets/adw-factory.json` that both append to the
  same file, merge one, run `adw run`; the other's PR goes `DIRTY → CLEAN`
  without an operator touching git, and its journal has the `rebase` event.
