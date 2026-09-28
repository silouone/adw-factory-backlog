---
id: adw-store-02-the-ledger-absorbs-an-operator-edit-cf9091
type: feat
status: blocked
priority: 2
created: 2026-09-28
review: false
caps: {minutes: 180, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-store-02-the-ledger-absorbs-an-operator-edit-cf9091-1790621998954","branch":"adw/adw-store-02-the-ledger-absorbs-an-operator-edit-cf9091","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-store-02-the-ledger-absorbs-an-operator-edit-cf9091-1790621998954/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# A hand-edited ticket does not need a manual commit before it can be dispatched

> Operator decision 2026-09-27: *"the central ledger, something is annoying
> because it needs commit — can we automate so the ledger is ALWAYS up to date,
> local and remote in sync?"* Measured the same evening: the ledger
> (`~/adw/backlog`, remote `silouone/adw-factory-backlog`) **already**
> auto-commits every factory transition — `adw: <id> → in-progress`, `→ done`,
> `→ in-review (PR #137)`. Two gaps remained: **push** (now closed by a
> `post-commit` hook, see §Already done) and **operator edits**, which is this
> ticket. Evidence: commit `a97e430 "wip"` is a hand commit the operator had to
> make purely so four hand-edited tickets could be dispatched.
> `review: false`: hard-gated.

## The friction

`commitTicketFile` (`src/intake/repo-commit.ts:239-241`) refuses when the
dispatched ticket's own file is dirty:

```ts
if (gitRunner(deps.repoPath, ["status", "--porcelain", "--", rel]) !== "") {
  throw deps.onDirtyTicket(rel);
}
```

So editing a ticket — flipping `status: blocked` → `queued`, adding a
constraint, or **writing a brand-new ticket at all** (untracked counts as dirty)
— makes the next dispatch refuse until the operator commits by hand. That hand
commit is pure ceremony: the factory is about to commit the same file one line
later.

Note what the guard is **already** narrow about: the pathspec `-- rel` scopes
both the check and every commit to that one ticket file. An edit to a *different*
ticket never trips it, and can never be swept into a transition commit. That is
what makes absorbing safe.

## What to build

When the only dirty path is the dispatched ticket's own file, **commit it as its
own commit** and continue, instead of refusing.

- [ ] **R1** Replace the `onDirtyTicket` throw with an **absorb step**: a commit
      of `rel` alone, landed **before** the caller's steps, **inside the same
      repo-lock acquisition**. It must NOT be a second `commitTicketFile` call —
      the module's own REENTRANCY COROLLARY (`repo-commit.ts:26-33`) forbids
      that and it would deadlock on its own `_repo.lock`.
- [ ] **R2** Message: `ledger: operator edit to <ticketId> (absorbed by run
      <runId>)`. `deps.runId` is already on `CommitWindowDeps`, so the edit
      stays attributable in `git log` and is never confused with a factory
      transition.
- [ ] **R3** **Untracked is the common case, not the edge.** A brand-new
      hand-written ticket reports `?? <rel>` — `git commit -- <rel>` alone will
      not include it. The absorb step must `git add -- <rel>` first.
      (Measured: `adw-gates-07-…-ad75a0` was exactly this.)
- [ ] **R4** `onDirtyTicket` is **kept**, not deleted — it stays the loud
      refusal for what absorption must not paper over (Art. IX):
      a merge-conflicted file (`UU`/`AA`), and any state where the add or the
      commit itself fails. Name the porcelain status code in the error.
- [ ] **R5** **Loud, not silent.** Absorbing prints a notice naming the ticket,
      the porcelain status it absorbed, and the resulting short sha. A guard that
      quietly stops guarding is worse than the ceremony it replaced.
- [ ] **R6** `kind: "plain"` stores are **byte-identical** — they skip the branch
      and dirty guards entirely today (`repo-commit.ts:202-205`) and must
      continue to invoke no git subprocess at all.

## TDD (Art. I — non-negotiable)

Tests first, red, reviewed, then green:

- a **modified** ticket → dispatch succeeds; **two** commits land, absorb first,
  transition second, both pathspec-limited to that one file;
- an **untracked** ticket → same (R3);
- **another ticket dirty at the same time** → it is still dirty afterwards and
  appears in **neither** commit — this is the guard's real purpose and the test
  that proves absorption did not widen it;
- a **merge-conflicted** ticket → still refuses through `onDirtyTicket` (R4);
- a **plain** store → the existing "throws on any git call" fake still sees zero
  invocations (R6);
- the lock-ordering and reentrancy suites stay green **unmodified**.

## Already done — do not rebuild

The **push** half is closed and needs no code: `~/adw/backlog/.git/hooks/post-commit`
backgrounds a `git push` after every commit, so the factory's four commit edges
and operator commits alike reach the remote. Verified 2026-09-27: it pushed
`3410092` and logged to `.git/push.log`; the remote's `osxkeychain` helper
authenticates even under `env -u GITHUB_TOKEN -u GH_TOKEN`, which is how every
run is launched. The hook deliberately does **no** `pull --rebase` — rewriting
history while the factory reads HEAD is how a ledger loses a transition.

## Out of scope

- **The hook is local-only and unversioned.** Moving it to a tracked `hooks/`
  dir plus `core.hooksPath` so it survives a re-clone is a separate ticket.
- A filesystem watcher that auto-commits every change: **rejected** by the
  operator's own decision. It breaks the single-writer rule `commitTicketFile`
  exists to enforce — it can commit a half-written ticket mid-transition and
  contend on `.git/index.lock` with the factory.
- Any change to the branch guard (N6) or to the dispatch/repo lock ordering.
