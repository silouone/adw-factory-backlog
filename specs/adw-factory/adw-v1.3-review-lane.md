# Amendment v1.3 — an agent reviewer between green gates and the PR

> **Status:** PROPOSED 2026-09-12. Not approved; not scheduled. Written from an
> operator observation: the three lanes have no stage whose job is to *judge*
> the work, only stages that produce it and a `gates` node that runs it.
> **Amends:** `adw-v1.md` §2 (non-goals) and §3 Story 1/2 acceptance; adds one
> agent node *type*, so `adw-v1-plan.md` §2 **Gate III is amended** (see §2).
> **Binding:** `constitution.md` — unchanged, and see §1 on Art. IV.

## 1. Trigger — why this is allowed, not a silent diverge

**Art. II — "not until a run demonstrates the need."** The need is demonstrated
three times over, in writing, by our own specs:

`adw-v1.1-lanes.md` §Residual Risks names three semantic holes and assigns every
one of them to the same backstop:

> - **`feat`'s test agent could author tests that pass trivially** rather than
>   meaningfully hardening coverage. Backstop: *operator review at merge*.
> - **A `gates ↻ repair` round could weaken/delete the bug's reproducing test**
>   to force green. Backstop: *operator review at merge*.
> - **Clean-red is structural, not semantic** — proves a well-formed test went
>   red then green, not that it reproduces *the* bug. *Prompt + review close the
>   gap.*

`tickets/BACKLOG.md` #5 concedes the consequence: *"Our architecture already
concedes review is the constraint (Art. IV puts the human exactly there)."*

And the practice already exists — the two-axis `/code-review` has gated **seven**
milestone exits by hand (`adw-m3`, `m4-03`, `m4-04`, `m4-06`, `m5-02`, `m5-03`,
`m6-04`). This amendment does not invent a workflow. It moves a proven,
hand-run, operator-executed step inside the machine.

**Art. IV is NOT amended and NOT violated.**

> *"A human writes the ticket; a human merges the PR. Everything between is
> agents + code."*

A reviewer node sits in **the middle**, which Art. IV assigns to agents. It
never merges, never pushes, never edits ticket intent. Story 3's human review
gate is untouched — it receives better input. Accountability is unmoved: the
agent's verdict is *evidence for* the human, never a substitute.

**What this is not.** Not a merge gate, not an approval authority, not a
quality score. A reviewer that can block a PR from opening would put an
unaccountable agent at the end; a reviewer that only *reports* keeps Art. IV
intact. See decision D3.

## 2. Gate III is amended — a third agent node type

`adw-v1-plan.md` §2 Gate III caps the system at **two** agent node types:
`build` (fresh query capturing a summary) and `repair-resume` (session-resuming
query). v1.1 deliberately did **not** amend it, because `plan`/`test` are
instances of `build`.

**A reviewer cannot be an instance of `build`.** `build` mutates the workspace
and returns a summary; a reviewer must be **read-only over a diff** and return a
**structured verdict**. Making it a `build` instance would give a reviewer write
access to the code it is judging — the exact conflict of interest the node
exists to remove.

So this adds **`review`**, a third type: *a fresh query over a read-only diff,
returning typed findings, with no workspace mutation.* Write-denial must be
enforced by the node, not by the prompt (see R4).

Gate III becomes: **three** agent node types.

## 3. Scope

### 3.1 Where it sits

    … build → test → gates (↻ repair ≤3) → REVIEW (↻ build-fix ≤N) → commit → push → open-pr

**After green gates, before commit.** Rationale: reviewing a red workspace wastes
tokens on findings the repair loop is already fixing, and reviewing after
`open-pr` means the human sees the PR before the machine's opinion of it.

Applies to all three lanes. `chore` is in scope but see D2.

### 3.2 The two axes (ported from the operator's `code-review` skill)

Two **parallel sub-reviews**, never merged, never re-ranked — the separation is
the point, because a change can pass one and fail the other:

- **Standards** — does the diff follow this repo's documented standards, plus a
  fixed Fowler smell baseline? Documented repo standards **override** the
  baseline; baseline smells are always judgement calls; skip anything tooling
  already enforces (that is `gates`' job, and duplicating it wastes the round).
- **Spec** — does the diff faithfully implement *the ticket*? Three questions:
  requirements missing or partial; behaviour present that was never asked for
  (scope creep); requirements that look implemented but wrong.

The factory has a decisive advantage the skill lacks: **the spec source is not
ambiguous.** The skill must hunt for a PRD; here the ticket *is* the contract,
and for `feat`/`bug` the plan artifact is already on `ctx.data`.

### 3.3 Feedback loop

A reviewer that only reports is a better PR body. A reviewer that can **return
work to the builder** is the actual ask. On findings above a severity floor, the
node routes to a fix stage with the findings as input, bounded by a hard ceiling
(Art. V). Unfixed findings below the floor ride into the PR body.

## 4. Requirements

**R1 — the `review` node type.** A fresh agent query over a read-only diff,
returning a typed verdict. No workspace mutation. Named stages are instances of
it, as `plan`/`test` are of `build`.

**R2 — two axes, parallel, unmerged.** Standards and Spec run as separate
queries and are reported separately. No single aggregate score; no re-ranking
across axes.

**R3 — a typed verdict, not prose.** Each finding carries at minimum: axis,
severity, file/hunk, the cited standard or ticket line, and whether it is a hard
violation or a judgement call. Free-form prose cannot drive a routing decision,
and this must be machine-readable to gate the loop in R5. *(This is roadmap 2.1
— typed agent envelopes — becoming load-bearing rather than speculative.)*

**R4 — write-denial is enforced, not requested.** The reviewer's tool allowlist
must exclude workspace mutation. A prompt saying "do not edit" is not a control;
the run must be *unable* to.

**R5 — bounded feedback loop (Art. V).** Findings at or above the severity floor
route to a fix stage with the verdict as input. A hard ceiling in code; breach →
stop, mark blocked, hand over the transcript. No unbounded review↔fix ping-pong.

**R6 — the verdict reaches the human.** Findings — fixed and unfixed — appear in
the PR body, attributed to the axis and severity. This is what converts review
from a cost into a briefing, and it is the measurable win for BACKLOG #5.

**R7 — journaled as a first-class node (Art. VI).** `node-start`/`node-end` with
a typed `details` bag carrying the verdict, so "what did the reviewer object to
and what happened next" is answerable from artifacts alone.

**R8 — per-ticket opt-out.** Frontmatter must be able to skip review, for the
same reason `caps:` exists. A one-line typo fix should not pay a review round.

## 5. Decisions the operator must make before any code

**D1 — severity floor for the feedback loop. DECIDED 2026-09-12:** work is sent
back for **Standards-axis HARD violations** (a documented repo rule breached)
and **Spec-axis "requirement missing or partial"**. Judgement calls (including
every baseline smell) and scope-creep notes do NOT route — they ride into the PR
body for the human. The floor is a rule in code, not the reviewer's opinion
(Art. III).

**D2 — does `chore` get a reviewer?** `chore` is "mechanical work, no judgment."
Reviewing it may be pure cost — or exactly where a bored agent cuts corners.

**D3 — can review block the PR entirely?** Proposed answer: **no** — findings
ride into the PR body and the human decides. A blocking reviewer puts an
unaccountable agent at the end (Art. IV). Worth stating explicitly, because the
opposite is tempting.

**D4 — model and cost.** Review adds ≥2 agent queries per run. Today the model
knob is per-run, not per-role — a reviewer probably wants a stronger model than
the builder. That pressures roadmap 2.6 (per-node agent roster) into being a
dependency rather than a nice-to-have.

**D5 — does the reviewer see the plan artifact?** For `feat`/`bug` it is on
`ctx.data`. Richer spec-axis context, but it also lets the reviewer inherit the
planner's blind spots — it is judging against the plan rather than the ticket.

**D6 — self-review on the self-target.** When the factory reviews its own code,
the reviewer reads the standards it is itself built on. Probably fine; worth
naming before it surprises someone.

## 6. Success metrics

- On a run where the builder shipped a real defect, the reviewer names it —
  measured against a deliberately-seeded defect, not a hypothetical.
- Findings appear in the PR body of every reviewed run, with axis + severity.
- Review→fix rounds have a journaled ceiling and no run exceeds it.
- **The one that matters (BACKLOG #5):** operator review minutes per PR fall.
  If they don't, this added cost and changed nothing — kill it.

## 7. Explicit non-goals

- **A merge gate.** See D3 and Art. IV.
- **A quality score or grade.** Findings are evidence, not a number.
- **Replacing `gates`.** Deterministic checks stay deterministic (Art. III —
  code before agents). The reviewer must *skip* what tooling enforces.
- **Replacing human review.** Story 3 is unchanged.
- **A third-party-tool reviewer.** Today's lesson: a gate whose failure
  implicates something other than the change under review is a liability.

## 8. Open question the operator flagged and this spec does not answer

> *"an upping review that can trigger a new build to take on feedback"*

Whether the fix stage is a **new build** (fresh context, sees only the findings)
or a **resumed session** (`repair-resume`, full context, may re-litigate its own
choices) is genuinely undecided. Resume is cheaper and already exists; fresh
context is likelier to actually take the feedback rather than defend the work.
This is the same tension as `bug`'s `build-fix RESUME`. **D7.**

**DECIDED 2026-09-12: a FRESH build that sees only the findings.** Not a
resumed session. The agent that wrote the code is the worst candidate to judge
whether the criticism lands — resume invites it to re-litigate its own choices
rather than act on them, and this is the one node whose entire purpose is to
act on someone else's judgement. It costs a fresh context load; that is the
price of not anchoring. Mechanically this makes the fix stage an instance of the
EXISTING `build` node (like `plan`/`test`), so only ONE new agent node type is
added overall — `review` — and Gate III goes 2 → 3, not 2 → 4.

## 9. Amendment (adw-bug-38, Art. XI): an absent required field is not a contract violation

`adw-bug-12` R3 — *"tolerance applies to the envelope, never the contract"* —
was pinned with an **inadmissible value** (`severity: "critical"`). A **missing
key** was never in that evidence, and treating the two alike left the bounded
re-ask loop (`REVIEW_PARSE_MAX_ATTEMPTS`) dead for every non-JSON-level error.
The contract class splits in two:

- **A required field absent** — a formatting slip of the same family as a
  markdown fence: retryable within the shared `REVIEW_PARSE_MAX_ATTEMPTS`
  ceiling, re-asked with the absent field path(s) and their admissible values
  named back to the agent. Exhaustion still blocks.
- **A field present with an inadmissible value, an axis mismatch, or an
  unknown field** — a genuine contract violation, terminal on the first
  attempt: the agent said something specific and wrong, and a re-ask invites it
  to guess. One such error anywhere in the verdict makes the whole verdict
  terminal (no partial salvage), even alongside absent fields.

The strict contract itself is unchanged: an absent `severity` is never
defaulted (that would let the parser invent a routing decision). The journal
distinguishes the classes: a verdict that parsed after a re-ask carries
`parseRetries` (`envelope` | `missing`, with the named fields) on its `review`
node-end; a terminal failure is told apart by its `reason` ("parsed but
invalid" vs "parse retries exhausted … missing required field(s)").
