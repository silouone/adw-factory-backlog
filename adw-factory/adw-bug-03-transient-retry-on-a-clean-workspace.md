---
id: adw-bug-03-transient-retry-on-a-clean-workspace
type: bug
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-bug-03-transient-retry-on-a-clean-workspace-1789407232962","branch":"adw/adw-bug-03-transient-retry-on-a-clean-workspace","workspace":"/Users/silouane/adw-factory/runs/adw-bug-03-transient-retry-on-a-clean-workspace-1789407232962/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/43","provider":"claude","model":"sonnet"}]
---
# A mid-stream stall on `build` throws away the run — even when the workspace is provably untouched

> Reproduced 2026-09-14 by `adw-m9-04-review-fix-loop-1789402361952`:
> **37 minutes and a complete plan lost to a stall that wrote nothing.**

## 1. What happened

```
plan  ↻ transient-retry 1/2  →  plan ✓  (35 turns, 42,744 output tokens)
build →  API Error: Response stalled mid-stream  →  run-end blocked
```

`adw-resilience-01` built exactly the machinery for this. It caught the stall
on `plan` and re-ran it cleanly. Then `build` hit the **same** stall and the
run died, because only `plan` opts in:

```ts
// feat.ts / bug.ts — plan only
makeBuildNode(deps.query, agentConfig, { name: "plan", retryTransient: true })
```

## 2. The stated reason, and why it did not apply here

`feat.ts` says it plainly, and the reasoning is **correct as far as it goes**:

> plan is READ-ONLY — its output lands only on ctx.data.plan, the workspace is
> untouched — so a dropped stream is safe to retry via a fresh session. Every
> OTHER agent stage (build, test) writes files; they keep today's
> is_error → fail, unchanged.

The premise is "build writes files, so a restart is not safe". **On this run
build wrote nothing.** Verified directly against the surviving worktree:

```
$ git -C runs/adw-m9-04-review-fix-loop-1789402361952/workspace status --short
(empty)
$ git -C … log --oneline main..HEAD
(empty)
```

The stall landed before the first `Edit`. So the run was thrown away under a
safety rule whose precondition was false — and that precondition is **cheap and
deterministic to check**, which is the whole finding.

## 3. Requirements

- [ ] A transient-shaped failure on a **workspace-writing** stage (`build`,
      `test`, `build-test-only`, `build-fix`, `repair`, `review-fix`) becomes
      retry-eligible **if and only if the workspace is provably unchanged** —
      `git status --porcelain` empty AND no commits ahead of the base.
- [ ] **Dirty workspace → today's behaviour, unchanged.** `fail`. Do not
      attempt to reconcile a partial edit; that is a different and much larger
      ticket, and guessing there is worse than blocking.
- [ ] The check is **deterministic and free** — a `git` call, no agent, no
      tokens. It belongs beside the existing `isTransientResultError`
      classifier, not inside an agent prompt.
- [ ] Same bound as today: `TRANSIENT_RETRY_MAX_ROUNDS` (2). Reuse it
      (Art. VIII); do not introduce a second ceiling.
- [ ] **Journal the decision either way.** "Stalled, workspace clean, retrying"
      and "stalled, workspace dirty, blocking" must both be answerable from
      `journal.jsonl` alone (Art. VI). A retry that is invisible is a retry
      nobody can audit.
- [ ] The restart is a **fresh session with the same prompt**, exactly as
      `makeTransientRetryNode` already does for `plan`. Not a resume — the
      stalled session's state is precisely what is not trustworthy.

## 4. Red tests

- [ ] A fake query that returns a `stalled mid-stream` result on the first
      `build` call and succeeds on the second, with a **clean** fake workspace
      → the run reaches `next`, and exactly one `round` event names `build`.
- [ ] The same fake, with a **dirty** workspace → `blocked` on the first
      stall, zero retries. Both directions, or the fix is a licence to
      double-apply edits.
- [ ] The clean/dirty probe is unit-tested against a real temp git repo — it
      is the predicate the whole fix rests on.
- [ ] Red before any fix.

## Verify

- [ ] `adw-m9-04-review-fix-loop` re-dispatches and survives a build-stage
      stall without losing its plan.
- [ ] A genuinely partial build still blocks, and says why.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## What this cost, so the priority is arguable from evidence

| | |
|---|---|
| wall clock lost | **37 minutes** (2,233,088 ms) |
| planning work lost | 35 turns, 42,744 output tokens, 2.06M cache reads |
| files the stall had written | **0** |

The plan itself was recoverable by hand from
`runs/<runId>/prompts/0003-build.txt` (it carries the `{{plan}}`
substitution) and is saved at `ai_docs/2026-09-14-m9-04-recovered-plan.md`.
**That recovery was manual and nothing in the factory offers it** — worth its
own ticket if this recurs.

## Out of scope

Resuming a *partially written* workspace. Persisting `ctx.data.plan` across
runs so a re-dispatch can skip planning — related, tempting, and a different
ticket.
