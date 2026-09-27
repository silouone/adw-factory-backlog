---
id: adw-bug-10-a-stall-must-resume-not-restart
type: bug
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-bug-10-a-stall-must-resume-not-restart-1789485517135","branch":"adw/adw-bug-10-a-stall-must-resume-not-restart","workspace":"/Users/silouane/adw-factory/runs/adw-bug-10-a-stall-must-resume-not-restart-1789485517135/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/55","provider":"claude","model":"sonnet"}]
---
# Any error that a retry could fix must be retried — today two regexes decide, and everything else throws the run away

> Operator, 2026-09-15: *"we can't have runs failing because of stalled
> mid-stream issues."* Agreed. Today three separate mechanisms each decline to
> handle it, for individually-good reasons, and the run dies in the gap.

## 1. The hole, in one run

`adw-bug-07-…-1789465894523` — the ticket that fixes the stall's root cause —
died to the stall. Bug lane, and it got **all the way through `red-check`**:

```
plan ✓ → build-test-only ✓ → red-check ✓ → build-fix ✗ stalled mid-stream
```

The worktree held **147 lines of reviewed red tests**, four failing for the
right reason, 90 pre-existing still green, lint and tsc clean. Then the run was
thrown away. Salvaged by hand onto
`adw/adw-bug-07-plan-and-build-must-write-their-artifacts` (commit `896777f`).

**Why nothing caught it:**

| mechanism | why it declined | correct? |
|---|---|---|
| `adw-resilience-01` transient-retry | only `plan` sets `retryTransient: true` | yes, as designed |
| `adw-bug-03` clean-workspace retry | the workspace was **dirty** (the tests) → refuse | yes, as designed |
| `adw-bug-05` | shipped §5 (journal the reason) only | the fix was never built |

Every one of those is individually defensible. Together they mean **a stall
after the agent has written anything is unrecoverable.** That is precisely the
moment when the work is most worth keeping.

## 2. Why "restart" is the wrong primitive, and resume is the right one

`makeTransientRetryNode`'s own doc comment says what it does:

> the re-run starts a **NEW session with the SAME prompt** — the RESTART
> semantic, not resume.

Restart is safe only for a read-only node, which is exactly why
`retryTransient` was scoped to `plan`. A restart on a node that already wrote
files risks double-applying work — `adw-bug-03`'s dirty-workspace refusal is
the honest consequence of that.

**But the bug lane already has the safe primitive.** `build-fix` resumes the
`build-test-only` session (`bug.ts:325`, `resume: sessionId`) precisely because
*"a cold restart would throw away the context in which the agent decided the
test's shape"* (README). A resumed session knows what it already wrote, so
continuing it is safe on a dirty workspace in a way a restart never is.

**The stall is a transport failure, not a reasoning failure.** The session on
the provider's side is intact; only the stream died. Resuming is the
semantically correct response.

## 3. The classifier is an allowlist of TWO strings — invert it

The operator, reading the first draft of this ticket: *"does this include ALL
TYPES OF ERROR, non-lenient — everything that CAN PASS if retried SHOULD be?"*
It did not. That is now the ticket's centre.

**The entire transient surface today:**

```ts
const TRANSIENT_RESULT_PATTERN = /stalled mid-stream/i;   // build.ts:991
/rate.?limit|429/i                                        // isRateLimit, :1107
```

Two regexes. **Everything else ends the run.** Measured across every banked
capture, the only error string that has ever actually appeared is
`API Error: Response stalled mid-stream` (~120 occurrences) — because it is the
only one anything survives long enough to record. And `ECONNRESET` appears in
exactly one place: a **test fixture that asserts a socket hangup KILLS the
run**:

```ts
expect(build.run(mkCtx())).rejects.toThrow("ECONNRESET");
```

A socket hangup is the textbook retryable failure, and we have a green test
pinning it as fatal.

**`adw-resilience-01` already asked for better and did not get it.** Its own
body says:

> **Do not classify on a guessed string list** — the `isRateLimit` regex is the
> precedent for the mechanism, not for guessing its contents.
> Split `is_error` into at least PERMANENT (auth, session invalid — ends the
> run) and TRANSIENT (transport dropped, upstream overloaded).

What shipped was one more regex. This ticket finishes that split.

### The inversion, and why it is the safe default

- [ ] **Enumerate PERMANENT, retry everything else.** Stop allowlisting
      transient errors; denylist the ones a retry provably cannot fix —
      auth/credential failure, invalid or expired session, malformed request,
      an explicit model refusal, and the engine's own ceilings (turn, wall
      clock, rounds, watchdog, hard-stop). Those keep today's behaviour exactly.
- [ ] **The default for an unrecognised error is RETRY.** The costs are wildly
      asymmetric and that is the whole argument: retrying a permanent error
      costs **two bounded rounds and still fails**; not retrying a transient one
      costs **the entire run** — 30+ minutes and, on this very ticket, a
      completed set of reviewed red tests. Bounded waste beats unbounded loss.
- [ ] **Known-transient shapes get named explicitly anyway**, so they are
      greppable and testable rather than merely falling through the default:
      stream stalled, connection reset / socket hangup, 429, 500/502/503/504,
      529 / overloaded, request timeout.
- [ ] **Delete or invert the ECONNRESET test.** A test asserting that a socket
      hangup kills the run is pinning the bug. It must become its opposite.
- [ ] **Journal the classification** — `permanent` vs `transient`, and which
      rule matched — so a wrong call is visible in artifacts rather than
      inferred from a terminal (Art. VI; `adw-bug-05` §5 landed the machinery).

## 3b. Resume, not restart — the mechanism the retry uses

- [ ] **On a retryable failure, if the failed node has a `sessionId`, RESUME
      it** rather than restarting. Reuse the existing seam (`bug.ts:325`'s
      `resume: sessionId`) — do not build a second mechanism (Art. VIII).
- [ ] **Extend `retryTransient` to every agent stage**, gated on the resume
      path rather than on workspace cleanliness: `build`, `test`,
      `build-test-only`, `build-fix`, `repair`, `review-fix`.
- [ ] **No `sessionId` yet** (the stream died before the session existed) → the
      workspace is necessarily untouched, so `adw-bug-03`'s clean-workspace
      restart applies unchanged. Keep it.
- [ ] **Resume + dirty workspace is allowed**, and must be tested explicitly —
      it is the case today's rules refuse, and it is the one that matters.
- [ ] **Bound it exactly as today** — `TRANSIENT_RETRY_MAX_ROUNDS` (2), reusing
      the constant. Something that fails twice more on resume is not transient.
- [ ] **Back off between attempts.** Three immediate retries into an overloaded
      upstream is how you stay overloaded — `adw-bug-09` makes the identical
      point about `push`/`open-pr`. Share one backoff helper (Art. VIII).
- [ ] **Journal each resume** distinctly from a restart.

## 4. This does not replace `adw-bug-07`

`adw-bug-07` removes the *cause* — agents stop emitting long final messages, so
the ~300s idle window closes. This ticket makes the *consequence* survivable
when a stall happens anyway, for any reason.

Both are worth having, and **there is a bootstrap argument for this one
first**: `adw-bug-07` has now failed to build because of the very stall it
fixes. A factory that cannot retry through a stall cannot build its own stall
fix.

## 5. Red tests

- [ ] A fake query that stalls on `build` **after** an `Edit`, with a
      `sessionId` present → the retry resumes (assert `resume` is passed on
      the options bag, mirroring m9-02's "assert on the options bag" stance),
      the workspace is not reverted, and the run continues.
- [ ] The same, with **no** `sessionId` and a clean workspace → restart, as
      `adw-bug-03` does today. Unchanged.
- [ ] The same, with no `sessionId` and a **dirty** workspace → still blocks.
      The one case that must stay refused.
- [ ] Two stalls in a row → blocked at `TRANSIENT_RETRY_MAX_ROUNDS`, reason
      naming the exhausted ceiling.
- [ ] **An `ECONNRESET` / socket hangup is RETRIED**, not thrown — the inverse
      of the test that exists today.
- [ ] **An unrecognised, never-seen-before error is RETRIED** (the default), and
      the journal says it was classified transient by fallback, not by a rule.
- [ ] **An auth failure is NOT retried** — it ends the run on the first
      attempt, exactly as today. This is the test that proves the denylist
      still bites.
- [ ] Red before any fix.

## Verify

- [ ] `adw-bug-07` dispatches and completes — it has now failed once on this.
- [ ] A stall mid-`build` on a dirty workspace no longer ends the run.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Related

`adw-bug-07` (remove the cause) · `adw-bug-03` (clean-workspace restart, which
this keeps for the no-session case) · `adw-bug-05` (the ~300s measurement and
the journal-the-reason machinery) · `adw-bug-09` (`push`/`open-pr` retry
without backoff — the same "a blip kills the run" shape at the network edge).
