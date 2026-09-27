---
id: adw-resilience-01-transient-stream-errors
type: feat
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-resilience-01-transient-stream-errors-1789347371327","branch":"adw/adw-resilience-01-transient-stream-errors","workspace":"/Users/silouane/adw-factory/runs/adw-resilience-01-transient-stream-errors-1789347371327/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/32","provider":"claude","model":"sonnet"}]
---
# A dropped stream ends the run, because every agent error is one bucket

> Minted 2026-09-14 from `adw-fe-08-live-sse-1789345184581`.

## Evidence

```
✗ adw-fe-08-live-sse blocked — node "plan": agent reported an error result:
  API Error: Response stalled mid-stream. The response above may be incomplete.
```

The run reached `plan`, the agent made **35 tool calls**, the stream dropped, and
the run ended `blocked`. The workspace had **zero** changed files — `plan` is a
read-only stage, so re-running it costs nothing and risks nothing.

The factory had everything it needed to recover and did not try.

This is the first occurrence in every banked journal (`grep` across all of
`runs/*/journal.jsonl` for `API Error|stalled|rate limit` matches this run
only), so it is rare — but it is also the cheapest possible class of failure to
survive, and today it costs a whole dispatch.

## Why it happens

`consumeAgentStream` (`src/pipeline/nodes/build.ts`) puts every error result in
one bucket:

```ts
if (message.is_error === true) {
  // E4 (adw-m4-03): auth/session failures arrive as error results —
  // fail gracefully into blocked instead of advancing to doomed gates.
  requestCleanup();
  return fail(`agent reported an error result: ${message.result}`);
}
```

That comment is right about what it was built for. "Not logged in" **must** end
the run — retrying it just burns the clock. But a stalled stream is not an auth
failure, and the two are indistinguishable here.

**The precedent for splitting already exists in the same function.** Rate limits
are classified out of the generic throw path:

```ts
function isRateLimit(error: unknown): boolean {
  return /rate.?limit|429/i.test(message_(error));
}
```

So "classify the message text, treat this class differently" is established in
this file, not a new idea.

## The machinery already exists too

The engine owns retries and rounds (Art. V — never a node):

```ts
const target   = node.retry?.target   ?? lane.repair;
const maxRounds = node.retry?.maxRounds ?? lane.maxRounds;
```

`adw-m8-02` added the per-node retry override precisely so one node could carry
its own target and ceiling (`red-check` ↻ `revise`, bounded at 2). A transient
transport failure is a third loop of the same shape. **Nothing new is needed in
the engine; the round counter must stay there.**

## The constraint that makes this non-trivial

**Nodes are not equally safe to re-run, and the ticket must say which is which
rather than assume.**

- `plan` is read-only — its output is an artifact on `ctx.data`, the workspace
  is untouched. Re-running is free.
- `build` / `test` / `build-fix` may have written files before the stream
  dropped. A fresh session re-entering a half-modified tree is a different
  proposition, and a *resumed* session may be unusable if the transport is what
  failed.

Do not paper over that difference. A retry that is safe for `plan` and wrong for
`build` is worse than no retry, because it will look like it works.

## Requirements

- [ ] Establish the failure shape first: what the SDK actually delivers on a
      stalled stream vs. an auth failure vs. an overloaded upstream. Record it.
      **Do not classify on a guessed string list** — the `isRateLimit` regex is
      the precedent for the mechanism, not for guessing its contents.
- [ ] Split `is_error` into at least PERMANENT (auth, session invalid — ends the
      run, today's behaviour, unchanged) and TRANSIENT (transport dropped,
      upstream overloaded).
- [ ] A transient failure gets a BOUNDED retry, with the round counter in the
      engine and the trip journaled — a silent retry is worse than a block
      (Art. VI).
- [ ] State explicitly, per agent node, whether it is safe to re-run from
      scratch, and only retry the ones that are. Say why for each.
- [ ] A permanent error must still end the run on the first occurrence. A
      regression test for "Not logged in" is the bar.

## Verify

- A faked stalled-stream result on `plan` retries and the run completes.
- A faked `Not logged in` result blocks on the first occurrence, no retry.
- The retry is visible in the journal — a reader can tell a retried run from a
  clean one.
- Rounds are bounded: a permanently stalling stream blocks, it does not spin.

## Out of scope

Resuming a dropped session rather than restarting it (the `build-fix` RESUME
path is the precedent, but a dead transport may make the session unusable —
separate question, separate evidence). Retrying the deterministic nodes, which
have their own failure semantics.
