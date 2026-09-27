---
id: adw-bug-13-the-review-loop-cannot-converge-and-one-flake-is-terminal
type: bug
status: done
priority: 1
review: false
created: 2026-09-18
depends: []
attempts: [{"runId":"adw-bug-13-the-review-loop-cannot-converge-and-one-flake-is-terminal-1789738387815","branch":"adw/adw-bug-13-the-review-loop-cannot-converge-and-one-flake-is-terminal","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-13-the-review-loop-cannot-converge-and-one-flake-is-terminal-1789738387815/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/78","provider":"claude","model":"sonnet"}]
---
# Three green runs, three blocked outcomes: the review↻fix loop cannot converge, and inside it a single flake is unrecoverable

> ## SPLIT 2026-09-18 — read this before closing
>
> This ticket was filed with four defects. **Only Defect B is fixed.** The
> rest are now tracked separately, so closing this ticket does not close them:
>
> | defect | state |
> |---|---|
> | **B** — a flaked gate inside `review-fix` is terminal | **DONE**, merged as PR #78 |
> | **0** — a `wrong` finding never reaches the fix agent | → `adw-review-01-route-and-block-on-what-matters` |
> | **0** — non-routed findings vanish silently | → `adw-review-02-unresolved-findings-ride-into-the-pr` |
> | **A** — rounds are blind redraws | → `adw-review-03-the-reviewer-must-see-its-last-round` |
> | **C** — `severity`/`kind` routing contradicts the spec | → answered by `specs/adw-v1.12-review-routing.md` §3, implemented by `adw-review-01` |
>
> `specs/adw-v1.12-review-routing.md` is the amendment that answers
> `adw-v1.3` **D3** — listed there as a decision the operator must make
> *before any code*, never made, and shipped the opposite way by default.
>
> **This ticket remains the evidence record.** The measurements below (the
> 44-finding breakdown, the prompt-level proof, the 1-green/9-blocked
> outcome) are what the three tickets above are justified by — do not delete
> them when this closes.

> `review: false` on this ticket, for the same reason `adw-bug-12` carried it:
> the lane being fixed is the lane that would judge the fix.
>
> This is the **third** occurrence of *the review lane discards completed work*.
> `adw-bug-12` (PR #73, merged 2026-09-17) fixed the **formatting** variant.
> These are the **substantive-disagreement** and **flake** variants, one day
> later.

## Evidence — three runs, 2026-09-18, all green at `gates`

| run | wall | cost | outcome |
|---|---|---|---|
| `adw-pr-01-…-1789714981882` | 86m 45s | ~$21.8 | blocked — `review-spec` exhausted 2 rounds |
| `adw-fe-15-…-1789714388413` | 118m 42s | ~$57.0 | blocked — `review-spec` exhausted 2 rounds |
| `adw-fe-22-…-1789719265397` | ~62m | ~$16.4 | blocked — `review-fix`: "the fix broke gates" |
| `adw-fe-24-…-1789738858464` | — | — | blocked — `review-spec` exhausted 2 rounds |

**Four runs of gate-green work reached `blocked`.** All four were salvaged by
hand as PRs #75, #76, #77 and #79 — every finding closed, every suite green,
nothing weakened.

Full analysis: `ai_docs/2026-09-18-review-loop-non-convergence.md`.

---

## Defect 0 — the fix agent is NEVER TOLD about a `wrong` finding (the root cause)

**This is the defect. Everything below is secondary to it.**

The question that exposed it: the reviewer identified a real bug in `adw-fe-24`
three times — so why did the build agent that ran immediately after never fix
it? Answer: **it was never asked to.** The finding never reached its prompt.

`verdict.ts`:

```ts
export function isBlocking(finding: Finding): boolean {
  if (finding.axis === "standards") return finding.severity === "hard";
  return finding.kind === "missing";      // "wrong" and "unasked" -> false
}
```

`fix-loop.ts` renders the fix agent's `{{findings}}` from
`unionBlockingFindings`, which is `…filter(isBlocking)`. So a spec finding of
kind **`wrong`** — the reviewer's own vocabulary for *"this is implemented, and
it does not match what the ticket asked for"*, i.e. **an actual bug** — is
raised, journaled, shown on the run screen, and then **silently dropped before
the prompt is built**. Same for `unasked` (scope creep).

Proof from the run's own prompts (`runs/adw-fe-24-…/prompts/`):

| prompt | findings handed to the fix agent | the `wrong` hash finding present? |
|---|---|---|
| `0006-review-fix.txt` (round 1) | 1 standards + 4 `missing` | **no** |
| `0009-review-fix.txt` (round 2) | 1 standards + 2 `missing` | **no** |

The build agent behaved correctly. It fixed every finding it was given, and was
never given the only one that mattered.

### Measured across every banked run

| spec finding kind | count | routed to the fix agent |
|---|---|---|
| `missing` | 30 | yes |
| `wrong` | 12 (4 `hard`) | **never** |
| `unasked` | 2 | **never** |

**14 of 44 spec findings — 32%, including every report of a real bug — were
discarded on the floor.** Not one `wrong` finding has ever reached a fix agent
in the history of this factory.

### The asymmetry that makes it worse

The two classes are treated in exactly opposite, exactly wrong ways:

- **`missing`** (overwhelmingly *"a test is absent"*) — always blocking, and
  drives the round counter to exhaustion.
- **`wrong`** (*"the code is incorrect"*) — never blocking, never routed,
  never fixed.

So the factory **blocks runs over absent tests while silently discarding
reports of actual defects.** `adw-fe-24` is the clean demonstration: it blocked
on missing-test findings while the genuine bug the reviewer had correctly
identified rode through untouched into a PR.

- [ ] Route `wrong` findings to the fix agent. A `hard`/`wrong` is the single
      most important thing a reviewer can say and is currently the one thing
      that cannot reach the builder.
- [ ] Decide `unasked` deliberately (it may well be non-blocking — but that
      must be a decision, not a side effect of the same predicate).
- [ ] Whatever the routing rule ends up being, **a finding that is not routed
      must be surfaced** — on the PR body at minimum. Nothing a reviewer
      raises may vanish silently.
- [ ] This changes `isBlocking`, which `specs/adw-v1.3-review-lane.md` D1
      pins. Propose the amendment first (see Defect C — same predicate, and
      they should be amended together).

## What this costs, measured

Every run that has ever reached the review lane:

| outcome | runs |
|---|---|
| green | **1** |
| blocked | **9** |
| no `run-end` | 2 |

**A 10% success rate.** Nine of ten completed runs through the review lane had
to be salvaged by hand. Before the lane was wired, `bun scripts/run-metrics.ts`
measured 38 green / 70 total overall and 9 / 17 on cLens. The review lane, as
built, is the factory's dominant failure mode — it is not catching bad work, it
is discarding good work while dropping the reports that matter.

## Defect A — review rounds are independent redraws, not iterations

`src/pipeline/nodes/review.ts` builds the reviewer's entire prompt as:

```ts
const basePrompt = assemblePrompt(
  ticket, target, instance.template, "", "", staged.diff,
).trimEnd();
```

Six arguments. The 7th `findings` slot — the one `makeReviewFixNode` passes in
`fix-loop.ts` — is **not passed here**. Each round's reviewer is a fresh agent
seeing only `ticketBody + conventions + contextPointers + diff`. It cannot know
a previous round raised this finding, that a fix agent considered it, or what
that agent answered.

Meanwhile `prompts/review-fix.md` tells the fix agent, in its definition of done:

> For **each** finding above: either fix it, **or state plainly, in your final
> message, exactly why it is wrong.**

**That second branch is implemented nowhere.** `makeReviewFixNode` returns
`{patch, usage, review}`, where `review` is the re-run *standards* verdict. The
rebuttal lands in the agent's summary text and goes no further. No waiver
channel, no carried-findings channel, no reviewer memory.

So whenever the fix agent does not change the diff, the loop is **provably
non-convergent**:

```
round 1: reviewer reads diff D  -> finding F ; fix agent cannot fix F, rebuts
round 2: reviewer reads diff D  -> finding F ; identical input, identical verdict
round 3: reviewer reads diff D  -> finding F ; rounds exhausted -> blocked
```

Both `adw-pr-01` and `adw-fe-15` show exactly this in their journals. The extra
rounds could not have helped — **`maxRounds > 1` is pure token burn on any
finding the fix agent won't or can't fix.**

There *is* a re-prompt mechanism in the same function (on a malformed envelope,
`currentPrompt` becomes the base prompt plus the bad output quoted back, bounded
by `REVIEW_PARSE_MAX_ATTEMPTS`). It carries malformed **output**, never a prior
round's **verdict**, and lives inside one node invocation. It does not rescue
the loop — but it proves the plumbing for a richer reviewer prompt already
exists.

### The sharpest evidence: `adw-fe-24`

The first two runs are sometimes dismissed as "the finding was unsatisfiable
from a sandbox." `adw-fe-24` removes that defence. **Every one of its nine
findings was fixable in the workspace, with no external resource.** What
happened instead:

- **Eight of the nine were already fixed** by the time the run died. The
  `drawer-attempt` live-tick assertion, the hash-mismatch and `ok:false`
  degraded-state tests, the deterministic-blocks test, and the
  `max-height`/`overflow` CSS bound all existed in the diff the *final*
  reviewer was reading. They were re-listed anyway.
- In one case the reviewer cited a neighbouring test (`server.test.ts:1090`)
  and concluded an assertion was absent, when the test asserting it sat a few
  lines below at `:1139`.
- **One finding was real and survived all three rounds unfixed:** the ticket
  requires a sidecar be "read AND hashed at most once per SSE connection", and
  `memoizeReadPromptFile` cached only the disk read, so a SHA-256 over a 41KB
  prompt still ran every tick. The reviewer identified it correctly, in the
  same words, three times. Nobody fixed it.

So the loop spent three rounds re-reporting solved problems while the one
genuine defect went through untouched — and then blocked the run anyway. A
reviewer that knew what the previous round had raised, and what was done about
it, could not have produced this outcome. **This is the argument for threading
findings forward, and it is stronger than either of the first two runs'.**

- [ ] Thread the prior round's blocking findings, and the fix agent's response
      to each, into the next reviewer's prompt via the existing 7th slot.
- [ ] A round must be able to end in "finding withdrawn" — the reviewer seeing
      a contested finding and accepting the answer.

## Defect B — `review-fix`'s gate check has no repair path, so one flake is terminal

`adw-fe-22` shows `gates` passing all three (`lint`, `typecheck`, `test`), then
minutes later, in the same workspace, review-fix's **in-process** gates re-check
failing with `gate "test" (bun run test) regressed, exit 1`. The run died.

**That suite passes on the committed state** — re-verified at 2381 pass / 4 skip
/ 0 fail during salvage. The failure does not reproduce, which is exactly what
the README already documents: *"the full suite is not deterministically green —
the container and E2B suites run against real Docker and real E2B and flake
under load."* Three runs were in flight concurrently.

The reason one flake was fatal is `src/pipeline/review/fix-loop.ts`:

```ts
const gatesResult = await makeGatesNode().run(ctx);
if (gatesResult.kind !== "next") {
  return fail(`… the fix broke gates — ${describeGateFailure(gatesResult)}`);
}
```

A `retry` from gates — the result that in the lane routes to `repair`, bounded
at 3 rounds — is **flattened into a hard `fail`**. The same flaky test costs a
retry in the lane proper and kills the run inside a fix round. The node's own
header claims this makes regression-catching "structural, not a race"; it also
makes nondeterminism unrecoverable.

- [ ] A `retry` from the in-process gates re-check must reach the repair path,
      not collapse to `fail`.
- [ ] Distinguish "the fix broke gates" from "a gate was flaky" — the node
      currently reports both identically.

## Defect C — `severity` is inert on the spec axis (needs a spec amendment first)

`src/pipeline/review/verdict.ts`:

```ts
if (finding.axis === "standards") return finding.severity === "hard";
return finding.kind === "missing";   // severity ignored
```

`adw-pr-01` was killed by a finding its own reviewer classified
`severity: "judgement"` — which `prompts/review-spec.md` defines as *"ambiguous,
stylistic, or a minor/edge-case deviation."* The reviewer has no vocabulary for
"a real `missing` gap that should not be terminal."

This is **deliberate** — `verdict.ts`'s header and `specs/adw-v1.3-review-lane.md`
D1 both state it. Per the amendment rule this cannot be changed in code here.

- [ ] Propose an amendment to `adw-v1.3` D1 so `severity` routes on the spec
      axis too. Do not change `isBlocking` before that amendment is accepted.

## The deeper fix, and why it is the right default (Art. IV)

Article IV puts humans at exactly two ends: writing the ticket, reviewing the
PR. **A reviewer agent that terminally blocks means the operator never reaches
the second end.** The README's own answer to the repair-weakening risk is
*"Read the diff on the PR; that is what the human review gate is for"* — and
`blocked` is precisely the outcome that denies the operator that diff.

`tickets/BACKLOG.md` candidate 3 already reaches the same conclusion from a
different direction: *"don't stop the run, steer the reviewer."*

- [ ] On the final round, carry still-blocking findings into the PR body
      (`prompts/pr-body.md` already renders from the journal) and let the run
      reach `open-pr` instead of `blocked`. The operator gets the diff, the
      verdict and the disagreement in one place, and decides.

This single change converts all three runs above from `blocked` into reviewable
PRs.

## Out of scope

- The environment gaps that *set up* two of these findings — `gh` not being on
  the agent tool allowlist, and `runs/` being unreachable from a sandbox.
  Separate ticket; see the analysis doc §4-D.
- Worktrees provisioned from a base missing the ticket's own spec (`adw-fe-22`).
  Separate defect, recorded in PR #77.

## Verify

- [ ] A red test proves Defect A: a second-round reviewer prompt contains the
      first round's finding and the fix agent's response to it.
- [ ] A red test proves Defect B: a `retry` result from the in-process gates
      re-check does not produce a terminal `fail`.
- [ ] A run whose final round still has a blocking finding reaches `open-pr`,
      and the PR body names the unresolved finding.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — all green.
