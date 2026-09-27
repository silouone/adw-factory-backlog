---
id: adw-perf-03-red-check-runs-the-whole-suite-to-watch-one-test-go-red
type: feat
status: done
priority: 1
created: 2026-09-16
depends: []
attempts: [{"runId":"adw-perf-03-red-check-runs-the-whole-suite-to-watch-one-test-go-red-1789588337797","branch":"adw/adw-perf-03-red-check-runs-the-whole-suite-to-watch-one-test-go-red","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-perf-03-red-check-runs-the-whole-suite-to-watch-one-test-go-red-1789588337797/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/59","provider":"claude","model":"sonnet"}]
---
# `red-check` executes 1832 tests to observe that **one** of them is red

> Measured 2026-09-16 on `adw-bug-09-the-network-edges-retry-without-backoff-1789510852817`
> (target `adw-factory`, worktree, sonnet), the first run with cLens capture
> live. 50 minutes of wall clock to fix a 6-line retry loop.

## 1. What was measured

Per-node, from `runs/<id>/journal.jsonl`:

| node | wall |
|---|---|
| `baseline` | 3.0 min |
| `plan` | 7.0 min (27 turns) |
| `build-test-only` | 6.3 min (18 turns) |
| `red-check` | 3.1 min |
| `build-fix` | 31 min+ (237 tool calls) |

Inside `build-fix`, from the cLens session capture, pairing `PreToolUse` →
`PostToolUse` on `Bash`:

```
320s   ← one call: the full gate suite
120s
 62s
 53s
------
10.3 min of the 31 min spent inside Bash
```

The repo's `test` gate is `bun run test` → `bun test --timeout=30000` →
**1832 tests across 72 files, ~300 s**.

## 2. The waste, named precisely

`build-test-only` writes **one** new test. `red-check` then has to answer a
single question: *did that one test go red, for the right reason?*

To answer it, it runs all 1832.

This is not the `gates` node being thorough at the end of a run — that is
correct and stays. This is the **red-first seam**, which runs immediately
after `baseline` has already proven the untouched checkout green, on a
worktree whose only delta is one added test file.

`src/pipeline/nodes/red-check.ts:12` is explicit that the run-all is
deliberate:

> `red-check` node (decision 7) — runs ALL gates UNCONDITIONALLY, collecting
> every result even past a failure (NOT the stop-at-first `gates` node:
> truncating before a `test` gate that is not last would let the classifier
> emit a false clean-red).

That reasoning is about **gate ordering** — do not stop early, or you may
never reach the `test` gate. It is *not* an argument that the `test` gate
must execute the entire suite. The two concerns have been conflated.

## 3. There is already a precedent for exactly this fix

`red-check.ts:19-25` records the last time this repo removed a redundant
full-suite exec from this very lane:

> The base-green check (decision 9) used to live here as its own node,
> re-running every gate on the untouched checkout right after `baseline` had
> already done exactly that (adw-gates-02, amendment 2026-09-14 in
> `specs/adw-v1.1-lanes.md`: **measured as a 16-minute duplicate full-suite
> exec**). It is now `makeBaselineGreenCheckNode` — PURE over the baseline
> `baseline` already recorded, no second gate exec.

Same shape, same lane, one node further along: a full suite executed to learn
something a narrower execution already establishes.

## 4. Second defect, larger than the first: the fix phase pays for the gate suite three times over

**The cheap-iteration guidance is being followed.** Measured on the same run,
pairing each `Bash` command with its captured `duration_ms`, the agent's
iteration loop is disciplined:

```
 12s   bun test test/pipeline/nodes/open-pr.test.ts
 17s   bun run test:unit
 53s   bun test test/cli.test.ts test/cli.observability.test.ts
 62s   bun test …/push.test.ts …/open-pr.test.ts …/ci-round.test.ts
120s   bun test test/pipeline/nodes/open-pr.test.ts
```

Targeted files, `-t` filters, `test:unit`. Exactly what the prompt asks for.
This half of the system **works** and must not be changed.

The waste is entirely in the three that follow:

```
320s   bun run lint && bunx tsc --noEmit && bun run test
298s   bun run test
293s   bun run lint && bunx tsc --noEmit && bun run test
```

**15.2 minutes of a 43-minute node**, spent re-running the same full gate
suite three times. `prompts/bug-build-fix.md:31-35` permits this exactly once,
and disclaims its value in the same breath:

> **Only when you believe you are done:** run the full gate command once.
> This is a courtesy check for your own benefit — the engine's own `gates`
> node re-runs the full suite unconditionally after your turn ends either
> way, **so your own run here is not the verification of record.**

By the prompt's own statement, all three runs are redundant: `gates` re-runs
everything regardless. The agent bought **zero** verification with 15.2
minutes. And running it three times rather than once suggests the loop is
*full-run → failure → edit → full-run*, with each cycle costing 5 minutes for
information a 17-second `test:unit` would mostly have carried.

This is a bigger prize than §2 (15.2 min vs 3.1 min) and is a separate fix:
§2 is structural (node behaviour), this is the prompt's fix-phase contract.
It likely deserves its own ticket — the minimal version is to stop inviting
the full gate run at all, since the node after it is the verification of
record.

## 5. What good looks like

The operator's framing, which this ticket adopts verbatim:

> It's fine to have a full-test baseline, and the red-first approach is great,
> but red-first **after a full green baseline** should only focus on the added
> test.

- `baseline` — **unchanged.** Full suite. It is what earns the right to narrow
  everything downstream.
- `red-check` — executes only the test(s) the `build-test-only` turn added,
  and classifies red from that. `classifyRed` stays a pure total function; only
  the gate execution it is handed gets narrowed.
- `build-fix` inner loop — the cheap loop becomes enforced or at least
  measured, not merely suggested.
- `gates` — **unchanged.** Full suite, unconditional, before commit. Nothing
  about the definition of done moves.

## 6. Open questions for the refinement pass

1. **How is "the test the turn added" identified?** Candidates: diff the
   worktree against `baseline`'s tree and take changed files under `test/`;
   or have `build-test-only` declare the path in its artifact. The first is
   deterministic and needs no agent cooperation — prefer it, per Art. III.
2. **Does narrowing weaken `classifyRed`?** A `broken-test` (red for the wrong
   reason, e.g. an import error) must still be distinguishable from a clean
   red. Confirm the narrowed execution still carries enough signal.
3. **Spec amendment required?** `specs/adw-v1.1-lanes.md` carries the
   2026-09-14 amendment describing the red-first seam. If narrowing changes a
   documented behavioral guarantee, **stop and propose the amendment** before
   implementing (CLAUDE.md, Art. amendment rule). Do not silently diverge.
4. **Is a scoped gate a target-config concern?** `targets/*.json` declares
   `test = bun run test`. A per-node scoped variant may belong in the target
   contract rather than hard-coded in the node.

## 7. Build protocol

TDD, Article I — no source before a reviewed, red test.

1. Write tests against the `red-check` seam proving:
   - a clean red on the added test alone still classifies `clean-red`;
   - a `broken-test` (red for the wrong reason) is still caught;
   - the narrowed execution does **not** run the full suite (assert on the
     commands issued through the injected `workspace.exec` seam — no real
     process, per the existing fakes in `test/pipeline/`).
2. Confirm them red. Present for operator review.
3. Implement to green.
4. `bun run lint && bunx tsc --noEmit && bun test` all green.

## 8. Expected payoff

Measured against `adw-bug-09-…-1789510852817` (50 min and counting):

| fix | saving |
|---|---|
| §2 — narrow `red-check` to the added test | 3.1 min → seconds |
| §4 — stop inviting the full gate run in the fix phase | **15.2 min** |

§4 is the larger prize by 5×, needs no new seam, and risks nothing: `gates`
already re-runs the full suite unconditionally, so the verification of record
is unchanged. Together they take a ~50-minute bug run toward ~30.

Neither touches `baseline` or `gates`. The definition of done does not move.
