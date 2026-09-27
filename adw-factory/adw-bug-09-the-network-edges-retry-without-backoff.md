---
id: adw-bug-09-the-network-edges-retry-without-backoff
type: bug
status: done
priority: 1
created: 2026-09-15
depends: [adw-bug-08-a-fail-reason-never-reaches-the-journal]
attempts: [{"runId":"adw-bug-09-the-network-edges-retry-without-backoff-1789510852817","branch":"adw/adw-bug-09-the-network-edges-retry-without-backoff","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-09-the-network-edges-retry-without-backoff-1789510852817/workspace","outcome":"done","pr":"https://github.com/silouone/adw-factory/pull/56","provider":"claude","model":"sonnet"}]
---
# `push` and `open-pr` "retry" three times in a tight loop — a blip lasting 8 seconds throws away the whole run

> Reproduced 2026-09-15 by `minio-quay-registry-move-1789464269318` (target
> `api-content`): **1m38s of gate-green work, blocked at the last network
> hop.** The identical `git push`, run by hand from the run's own untouched
> worktree ~20 minutes later, succeeded on the first attempt.

## 1. What happened

```
build ✓ (1m18s)  →  gates ✓ compose-config ✓ minio-registry  →  push ✕  →  blocked
```

Everything the run existed to produce was finished, committed and verified.
The branch never reached origin. The ticket was finalized `blocked`, and the
PR had to be opened by a human from the leftover worktree.

## 2. Root cause: three attempts with no delay is one attempt with extra TCP

`push.ts:206`:

```ts
let lastError = "";
for (let attempt = 1; attempt <= PUSH_ATTEMPTS; attempt += 1) {
  const push = await workspace.push(branch);
  if (push.code === 0) return { kind: "next", patch: { attemptBranch: branch } };
  lastError = push.stderr.trim();
}
return fail(`push failed after ${PUSH_ATTEMPTS} attempts (${lastError}) …`);
```

`PUSH_ATTEMPTS = 3`. There is **no sleep, no backoff, no jitter** between
iterations. `open-pr.ts:547` has the identical shape for `gh pr create`
(`PR_ATTEMPTS = 3`, no delay).

Measured on the failing run: `push` node-start → node-end was **8055 ms** for
all three attempts. So the entire tolerance this factory has for a transient
remote condition is **eight seconds**, consumed as fast as the network will
return three errors.

That this is the outlier, not the norm, is visible in the codebase itself —
every *other* bounded loop in the pipeline waits between tries:

| loop | delays? |
|---|---|
| `repo-commit.ts` repo lock | yes — `sleepSync(pollIntervalMs)`, bounded wait |
| `baseline.ts` baseline lock | yes — `BASELINE_LOCK_POLL_INTERVAL_MS` |
| `build.ts` transient-stream retry (`adw-resilience-01`) | yes |
| **`push.ts`** | **no** |
| **`open-pr.ts`** | **no** |

The two loops with no delay are exactly the two that talk to a remote host.

## 3. The hypothesis that was wrong, and why it matters

The first explanation offered was contention from seven parallel runs. **That
is false**, and the journals say so. Push windows across the seven-run sweep:

| target | push started | push ms | outcome |
|---|---|---|---|
| coorpacademy | +70s | 5312 | ok |
| **api-content** | **+158s** | **8055** | **fail** |
| coorpacademy-oplog | +221s | 9941 | ok |
| coorpacademy-lambda | +271s | 2993 | ok |
| serverless-plugins | +316s | 4057 | ok |
| bricklane-persona-hooks | +389s | 2226 | ok |
| translated-language-service | +651s | 2642 | ok |

No two push windows overlap — api-content's push was the only one in flight.
And `coorpacademy-oplog` **succeeded after 9941 ms**, longer than the 8055 ms
in which api-content exhausted all three attempts and gave up.

So a healthy push to these repos ranges 2.2s–9.9s, and the failing run's
entire retry budget (8.1s) sits *inside* that normal range. Whatever the
remote condition was, an 8-second ceiling could not outlast it.

**The actual git stderr is unrecoverable** — `adw-bug-08` is why, and is why
this ticket `depends:` on it. Fixing the reason-journaling first means the
next occurrence names itself instead of needing this forensics.

## 4. Scope

Give both remote-facing loops a real retry policy:

- **Backoff with jitter** between attempts, not a tight loop. The budget
  should be wall-clock-shaped (seconds, tolerating a blip an order of
  magnitude longer than today's 8s), not attempt-count-shaped.
- **Classify transient vs permanent.** A non-fast-forward reject, a missing
  ssh key, or `403 Forbidden` is not going to become true on attempt 2 — it
  should fail fast with its reason rather than burning the budget. A
  `503`/timeout/connection-reset/rate-limit should ride the full budget.
- **Journal each retry.** A `round`-shaped event per attempt, so
  `just fails <run>` shows "push attempt 2 of N — <reason>" instead of a
  single silent `fail`. Today a retried-then-succeeded push is invisible.
- **One representation, both sites** (Art. VIII). `push` and `open-pr` must
  not grow two hand-rolled backoff loops; extract the policy once.

Deliberately out of scope: making `blocked` recoverable (`adw sync` /
`ci-round` already own resumption of an existing attempt), changing
`PUSH_ATTEMPTS`/`PR_ATTEMPTS` as bare numbers without a delay, and any
liveness change to the agent stream (`adw-bug-03`, done).

## 5. Red test first (Art. I)

1. A `workspace.push` that fails with a **transient** stderr twice and then
   succeeds must leave the node `next` — and the injected clock must show the
   node waited between attempts. Red today: it retries with zero delay, and
   nothing asserts the wait.
2. A `workspace.push` that fails with a **permanent** stderr (non-fast-forward
   reject) must `fail` **immediately**, on attempt 1, without consuming the
   budget.
3. Every attempt after the first journals its own event carrying the attempt
   number and the reason it is retrying.
4. The same three properties hold for `gh pr create` in `open-pr`, driven
   through the same extracted policy — not a second copy of it.
5. A first-attempt success is byte-identical to today: no wait, no extra
   journal event.

## Verify

- `bun run lint && bunx tsc --noEmit && bun test` green.
- `just fails <run-id>` on a run whose push retried shows each attempt.
- A fake push failing transiently for longer than today's 8s ceiling now
  reaches green instead of blocked.

---

## Resolution — shipped 2026-09-15, PR #56

Run `adw-bug-09-the-network-edges-retry-without-backoff-1789510852817`
(worktree, claude/sonnet) — **green in 66m 15s**. Merged
2026-09-15T23:28:36Z.

A new pure seam, `src/pipeline/retry-policy.ts` (+141), replaces the tight
`for` loop this ticket reported. `push.ts` (+86/-) and `open-pr.ts` (+100/-)
now consume it instead of counting attempts inline; the three lane
definitions and `ci-round.ts` were threaded through. 870 insertions across
16 files, with `retry-policy.test.ts` (+117) covering the policy in
isolation — the backoff is now a total function of its inputs, testable
without a clock or a network.

> **Ledger note.** This ticket read `status: queued` / `attempts: []` for a
> day *after* PR #56 merged — the run's own finalizer never wrote its
> attempt back. Repaired by hand 2026-09-16 from the run journal
> (`run-end: green, 3975030ms`) and the PR's merge state. Anything selecting
> on `queued` would have re-dispatched merged work. The finalizer gap is a
> defect in its own right and wants its own ticket; `adw-bug-08`
> (a fail reason never reaches the journal) is the nearest relative.
