---
id: adw-review-03-the-reviewer-must-see-its-last-round
type: bug
status: done
priority: 2
review: false
created: 2026-09-18
depends: [adw-review-01-route-and-block-on-what-matters]
attempts: [{"runId":"adw-review-03-the-reviewer-must-see-its-last-round-1789893696922","branch":"adw/adw-review-03-the-reviewer-must-see-its-last-round","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-review-03-the-reviewer-must-see-its-last-round-1789893696922/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/97","provider":"claude","model":"sonnet"}]
---
# Review rounds are independent redraws, not iterations

> Was Defect A of `adw-bug-13`. Split out so it can be worked separately from
> the routing fix, which is the urgent one.

## The defect

`src/pipeline/nodes/review.ts` builds the reviewer's entire prompt as:

```ts
const basePrompt = assemblePrompt(
  ticket, target, instance.template, "", "", staged.diff,
).trimEnd();
```

Six arguments. The 7th `findings` slot — the one `makeReviewFixNode` uses in
`fix-loop.ts` — is **not passed**. Each round's reviewer is a fresh agent
seeing only `ticketBody + conventions + contextPointers + diff`. It cannot know
a previous round raised a finding, that a fix agent acted on it, or what that
agent answered.

Meanwhile `prompts/review-fix.md` tells the fix agent, in its definition of
done:

> For **each** finding above: either fix it, **or state plainly, in your final
> message, exactly why it is wrong.**

**That second branch is implemented nowhere.** `makeReviewFixNode` returns
`{patch, usage, review}`, where `review` is the re-run *standards* verdict. The
rebuttal lands in the agent's summary text and goes no further.

There *is* a re-prompt mechanism in the same function — on a malformed
envelope, `currentPrompt` becomes the base prompt plus the bad output quoted
back, bounded by `REVIEW_PARSE_MAX_ATTEMPTS`. It carries malformed **output**,
never a prior round's **verdict**, and lives inside one node invocation. It does
not rescue the loop, but it proves the plumbing for a richer reviewer prompt
already exists.

## What it costs

`adw-fe-24` is the clean case. Of nine findings raised across three rounds,
**eight had already been fixed** by the time the run died — the
`drawer-attempt` live-tick assertion, the hash-mismatch and `ok:false`
degraded-state tests, the deterministic-blocks test, and the
`max-height`/`overflow` CSS bound all existed in the diff the *final* reviewer
was reading. They were re-listed anyway. In one case the reviewer cited a
neighbouring test (`server.test.ts:1090`), concluded an assertion was absent,
and the test asserting it sat at `:1139`.

So the loop spent three rounds re-reporting solved problems. With no memory,
**`maxRounds > 1` is close to pure token burn**: an unchanged diff yields an
identical verdict, and a changed one gets re-judged from zero.

`adw-review-01` reduces the blast radius (fewer findings can block) but does
not fix this: a reviewer that cannot see its own last round will still churn.

## Requirements

- [ ] Thread the previous round's findings, and the fix agent's response to
      each, into the next reviewer's prompt — via the `assemblePrompt` slot
      that already exists.
- [ ] A round may end in **"finding withdrawn"**: the reviewer sees a contested
      finding, accepts the answer, and does not re-raise it.
- [ ] Give the fix agent's rebuttal a destination. `review-fix.md` already
      promises the agent it may argue instead of fixing; make that branch real
      — a typed `{finding, disposition: "fixed" | "wont-fix", reason}` carried
      out of the node and journaled.
- [ ] A withdrawn or `wont-fix` finding still reaches the PR body
      (`adw-review-02`) with its reason. Disagreement is evidence, not silence.

## Verify

- [ ] Red test: the round-2 reviewer prompt contains round 1's finding **and**
      the fix agent's response to it. RED today — the prompt is built from six
      arguments.
- [ ] Red test: a fix node that returns `wont-fix` with a reason journals that
      disposition rather than discarding it.
- [ ] A finding withdrawn in round 2 does not appear as unresolved in the PR
      body, but its withdrawal is journaled.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

- The routing/blocking predicate — `adw-review-01`.
- PR-body rendering — `adw-review-02`.
