# Amendment v1.12 — a reviewer may report anything; it may block almost nothing

> **Status:** proposed 2026-09-18, from operator decision on live evidence —
> every run that has reached the review lane, measured from banked journals.
> **Amends:** `adw-v1.3-review-lane.md` — **D1** (the routing floor) and
> **D3** (whether review may block), which was listed as a decision the
> operator must make *before any code* and was never made. The lane shipped
> the opposite of D3's proposed answer.
> **Binding:** `constitution.md` — unchanged, and Article IV is the reason
> this amendment exists.
> **Implemented by:** `tickets/adw-review-01-route-and-block-on-what-matters.md`,
> `tickets/adw-review-02-unresolved-findings-ride-into-the-pr.md`,
> `tickets/adw-review-03-the-reviewer-must-see-its-last-round.md`.
> **Evidence:** `tickets/adw-bug-13-…` and
> `ai_docs/2026-09-18-review-loop-non-convergence.md`.

## 1. What the evidence says

Every run that has ever reached the review lane:

| outcome | runs |
|---|---|
| green | **1** |
| blocked | **9** |
| no `run-end` | 2 |

Before the lane existed, `bun scripts/run-metrics.ts` measured 38 green of 70
runs overall, and 9 of 17 on cLens. **The review lane is now the factory's
dominant failure mode.** All nine blocked runs were correct, gate-green work
that had to be salvaged by hand as PRs #75, #76, #77 and #79.

And it is not failing because it is strict. It is failing because it routes
the wrong things:

| spec finding kind | raised, all runs | reached the fix agent |
|---|---|---|
| `missing` | 30 | yes |
| `wrong` | 12 (4 `hard`) | **never** |
| `unasked` | 2 | **never** |

**14 of 44 spec findings — including every report of an actual bug — were
discarded before the fix agent's prompt was built.** In `adw-fe-24` the
reviewer correctly identified a real defect three times; the build agent that
ran immediately after was never told, fixed everything it *was* told, and the
run blocked anyway over absent tests.

## 2. Where the contradiction is

`adw-v1.3` **D1** says work is sent back for *"Standards-axis HARD violations"*
and *"Spec-axis 'requirement missing or partial'"*, and that *"judgement calls
… and scope-creep notes do NOT route."*

D1 never mentions `wrong`. The `missing`/`unasked`/`wrong` taxonomy was
invented later, in `tickets/adw-m9-01-review-verdict-contract.md`, which
assigned routing to `kind: "missing"` alone — *"Everything else is `false`."*
`src/pipeline/review/verdict.ts` implements that ticket faithfully.

So the code is not buggy against its ticket; **the ticket decided something
the spec never authorised**, and the spec's silence became "a reported bug
never reaches the builder."

`adw-v1.3` **D3** — *"can review block the PR entirely? Proposed answer: **no**
— findings ride into the PR body and the human decides. A blocking reviewer
puts an unaccountable agent at the end (Art. IV)"* — was never answered, and
the implementation chose "yes" by default.

## 3. D3 — ANSWERED 2026-09-18: **review may block, but only on a confirmed real defect.**

Not the spec's proposed "never block", and not today's "block on anything
missing". The operator's decision:

| finding | routes to the fix agent | may block the run |
|---|---|---|
| `standards`, `severity: hard` | yes | **yes** |
| `spec`, `kind: wrong`, `severity: hard` | yes | **yes** |
| `spec`, `kind: missing` (any severity) | yes | no |
| `spec`, `kind: wrong`, `severity: judgement` | yes | no |
| `spec`, `kind: unasked` | no | no |
| `standards`, `severity: judgement` | no | no |

Two distinct notions, conflated in the current code and separated here:

- **Routing** — does this finding reach the fix agent's `{{findings}}`? A
  finding that routes gets a chance to be fixed cheaply, while the workspace
  and the agent are already warm.
- **Blocking** — may the run's terminal outcome be `blocked` because this
  finding survived? Only the two rows above.

**The round counter advances only while a *blocking* finding is outstanding.**
A round in which every remaining finding is advisory ends the loop and the run
proceeds to `open-pr`.

### 3.1 Amendment 2026-09-27: a `missing`/`hard` finding opens one round

**Operator decision, 2026-09-27**, on evidence: a round opened only while a
*blocking* finding was outstanding, so a review whose only findings were
`spec`/`missing`/`hard` ended `next`. The reviewer prompt defines `missing` as
*"a requirement the ticket states is absent from the diff, or only partially
implemented"*, not only "a test is absent". Three green runs since #80 shipped
with unimplemented requirements and no fix round: `adw-bug-21` (R2–R5 absent,
5× `missing/hard`), `adw-cost-02` (#106, 2×), `sabado-14` (1×).

The rule, replacing the bold sentence above for this one case:

- A `spec`/`missing`/`hard` finding **opens a review-fix round** when no round
  has run yet, even if nothing blocking is outstanding. It routes as before.
- It **still never blocks.** If it survives that round and nothing blocking
  remains, the loop ends and the run proceeds to `open-pr` with the finding
  annotated (§4).
- `missing`/`judgement` is unchanged: it routes with a round opened by
  something else, and never opens one itself.

Implemented by `tickets/adw-review-06-a-missing-requirement-opens-a-round.md`.

### Why this shape

*"The code is wrong"* is the one thing a human reviewer cannot cheaply
re-derive from the diff — it needs the ticket, the surrounding code and an
argument. *"A test is absent"* is visible in the diff at a glance. So the
factory should spend its veto on the former and annotate the latter.

Against the four salvaged runs, this rule predicts:

- `adw-fe-24` — blocks. Correct: a `hard`/`wrong` defect that really was in
  the diff, and really did ship.
- `adw-fe-15`, `adw-pr-01` — proceed to `open-pr` with their `missing`
  findings annotated. Correct: both findings needed a resource no sandboxed
  agent had, and both diffs were sound.
- `adw-fe-22` — proceeds (its blocker was a flaked gate, fixed separately by
  `adw-bug-13` Defect B, merged as #79).

## 4. Nothing a reviewer says may vanish

Today a non-routed finding is filtered out of `unionBlockingFindings` and never
appears again outside `journal.jsonl`. D1 already promised the opposite —
judgement calls and scope-creep notes *"ride into the PR body for the human."*
That was never implemented.

**Every finding the reviewer raises, on either axis, of any kind or severity,
that is not resolved by the end of the loop, appears in the PR body**, tagged
with its axis, kind, severity and file, and marked as un-addressed. The
operator reads the diff and the objections together. That is what Article IV's
second end *is*.

## 5. What is NOT changed

- The two-axis split, one verdict per axis, never merged (R2).
- The typed verdict contract and its parser (`adw-m9-01`).
- `severity` remaining a required field on every finding.
- The fix agent staying a FRESH build (D7), never a resumed session.
- `REVIEW_FIX_MAX_ROUNDS = 2`.

## 6. Open, deliberately

- **D2** (does `chore` get a reviewer?) is untouched here.
- **D4** (a stronger model for the reviewer than the builder) is now partly
  served by `adw-profile-01`, merged as #74 — a per-stage model and effort
  exists. Whether to *use* a deeper profile for `review-spec` is a separate
  operator call.
