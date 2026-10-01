---
id: adw-bug-38-a-dropped-field-in-a-verdict-kills-a-green-run-7d6735
type: bug
status: queued
priority: 1
review: false
created: 2026-10-02
caps: {minutes: 120, turns: 250, stallMinutes: 20}
depends: []
attempts: []
---
# A reviewer that drops one required field kills a green run instantly, and the retry loop built for exactly this never runs

## The bug

Run `hq-01-buildgraph-is-pure-and-tested-0aabb3-1790893493810` (silou-hq, sonnet-5-5)
reached `plan → build → test → gates` with **lint, typecheck and 115 tests green**, then
died at `review-standards`:

```
verdict parsed but invalid — findings[0].severity: must be "hard" or "judgement"
(got undefined); findings[1].severity: …; findings[2].severity: … 
```

Run outcome `blocked`. 1994 insertions sat uncommitted in the run workspace until an
operator salvaged them by hand (silou-hq PR #1).

The reviewer's raw response (`artifacts/review-standards.md`) is otherwise **perfect**:
valid bare JSON, correct `axis`, three well-formed findings with `file`, `line`,
`citation`, `summary`. It simply never emitted the `severity` key. The tell is that it
reasoned about severity **in the wrong slot** — in prose, inside `summary`:

> "Moving it verbatim was reasonable for a no-behaviour-change ticket, **so this is a
> judgement call**."
> "This is **a baseline primitive-obsession smell rather than a rule breach**."

Findings 1 and 2 (long method, duplicated code) are baseline-catalog smells;
`prompts/review-standards.md` states those are *always* `"judgement"`, and a
`standards`/`judgement` finding neither routes nor blocks (`adw-v1.12` §3). Finding 3 is
**undecided by the reviewer itself**: its `citation` is
`"CLAUDE.md Craft: 'strict TypeScript'; ~/.claude/CLAUDE.md §5: 'strict typing'"` — the
kind-1 documented-rule form, the only kind permitted to be `hard` — while its `summary`
says the opposite, "a baseline primitive-obsession smell rather than a rule breach". So
whether this run would have routed a fix round is **not** establishable from the artifact,
and this ticket does not claim it.

That contradiction is the sharper evidence. The reviewer did not merely drop a key: its
citation and its summary disagree about which finding-kind it was looking at, which means
it never resolved severity at all. A re-ask that **names the slot and restates the
admissible values** (R3) is the fix that case needs. Any scheme that *defaults* an absent
`severity` would have silently invented a routing decision the reviewer had not made — see
"Unchanged" below.

## The fourth occurrence of one failure mode

`adw-bug-13` is the evidence record for *the review lane discards completed work*, and
names its variants: `adw-bug-12` (PR #73) fixed the **formatting** variant, `adw-bug-13`
the **substantive-disagreement** and **gate-flake** variants. This is the **dropped-required-field**
variant — a fourth way the same lane throws away a green run. No dependency on those
tickets (all `done`); this is their missing sibling, not a reopening.

`review: false` on this ticket, for the reason `adw-bug-12` and `adw-bug-13` both carried
it: the lane being fixed is the lane that would judge the fix.

## Why it happens (read before fixing)

`src/pipeline/nodes/review.ts:342`

```ts
function isEnvelopeFailure(errors: readonly VerdictValidationError[]): boolean {
  return errors.every((error) => error.field === "json");
}
```

A missing required field produces `field: "findings[0].severity"`, so
`isEnvelopeFailure` is false, the `!isEnvelopeFailure` branch at `review.ts:594` returns
`fail(...)` immediately, and `REVIEW_PARSE_MAX_ATTEMPTS` is **never consulted**. The
bounded re-ask loop that `adw-bug-12` R2 built for precisely this class of failure is dead
code for every non-JSON-level error.

This is `adw-bug-12` **R5 only half-delivered**:

> "A reviewer failure must never be more fatal than a reviewer finding. A blocking
> *finding* routes to a bounded fix loop and only then blocks. A malformed *verdict*
> currently blocks instantly, so the reviewer is harshest when it is least intelligible.
> After R2 the two paths are comparably bounded."

They are not comparably bounded. For an omitted key the verdict path is still instantly
terminal.

## The amendment this needs (Art. XI — do not skip)

`adw-bug-12` R3 reads: *"Tolerance applies to the envelope, never the contract."* Its TDD
pinned that with a finding carrying `severity:"critical"` — an **invalid value**. A
**missing key** was never in that evidence, and conflating the two is the defect. Propose,
in `specs/adw-v1.3-review-lane.md`, splitting the contract class in two:

- **A required field absent** → a formatting slip of the same family as a markdown fence:
  **retryable once**, re-asked with the absent field(s) **named back** to the agent.
- **A field present with an inadmissible value**, an axis mismatch, or an unknown field →
  a genuine contract violation: **terminal, as today**. The agent said something specific
  and wrong; re-asking invites it to guess.

Do not widen this to "retry everything". The distinction above is the whole fix.

## Requirements

- [ ] **R1 — classify the parse failure in three ways, not two.** `parseVerdict` already
      knows which fields were absent versus inadmissible, but flattens both into
      `{field, reason}` strings that `review.ts` must pattern-match. Carry the distinction
      as data on `VerdictValidationError` (e.g. a `kind: "envelope" | "missing" | "invalid"`
      discriminant) — pure, in `verdict.ts`, no string sniffing in the node.
- [ ] **R2 — a missing-field failure enters the existing bounded retry loop**, sharing
      `REVIEW_PARSE_MAX_ATTEMPTS` with the envelope case. Exhausting it still blocks.
- [ ] **R3 — the re-ask names the absent fields.** `reviewRetryPrompt` currently says only
      "could not be parsed (unparseable)". For this class it must say which fields were
      absent, at which finding indices, and restate the admissible values — a reviewer that
      put its severity in prose needs to be told the slot, not told to try again.
- [ ] **R4 — a mixed error list is terminal.** If any error is `"invalid"`, the whole
      verdict is a contract violation, even if other errors are merely `"missing"`. No
      partial salvage, no dropping of unparseable findings.
- [ ] **R5 — journal the class.** `node-end` details must distinguish
      missing-field-retried from parsed-but-invalid-terminal, so the next audit can count
      how often each fires instead of re-reading artifacts.
- [ ] **Unchanged:** the strict contract itself (no defaulting an absent `severity` to
      `"judgement"` — that would let the parser invent a routing decision); R1 artifact
      persistence before parsing; the S2.7 push guard; the prompts' "ONLY raw JSON" ask.

## TDD (Art. I — non-negotiable)

Red first, at the real `makeReviewNode` seam with a fake query — the same layer
`adw-bug-10` and `adw-bug-12` used.

- [ ] A fake query returning the **verbatim** `artifacts/review-standards.md` from
      `hq-01-…-1790893493810` (three findings, no `severity`): today it fails terminally on
      the first attempt. After the fix it is re-asked; a second response that includes
      `"severity":"judgement"` parses, nothing routes, and the node returns `next`. RED
      today.
- [ ] The same fake query returning a severity-less verdict on **every** attempt: the run
      blocks naming the ceiling, with every raw attempt persisted.
- [ ] Regression, must stay terminal and unretried: a finding with
      `"severity":"critical"` (inadmissible value — `adw-bug-12`'s own case); an axis
      mismatch; an unknown field.
- [ ] R4: one finding missing `severity` **and** one with `"severity":"critical"` →
      terminal on the first attempt.
- [ ] The re-ask prompt text contains the absent field path and the admissible values
      (assert on the prompt handed to the second query, not just on the outcome).
- [ ] Regression: a clean verdict is byte-identical to today, and a `hard` standards
      finding still routes to `review-fix` and still cannot reach `push`.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`.
- [ ] Re-run a silou-hq ticket (`hq-02` or `hq-09`) through a live lane with review enabled
      and reach `open-pr`.

## Out of scope — but file it

Moving the verdict off free text onto a **forced structured-output tool call**, which would
delete this failure mode rather than mitigate it. `adw-bug-12` already named this as the
better long-term answer and deliberately deferred it; this run is the second time the same
family has cost a green run, so the follow-up ticket should now be written (the SDK's
forced-tool-call path is what the `Workflow` harness already uses for its own schemas).
Also out: changing `isRouting`/`isBlocking`, and anything about what the reviewers look for
(that is `adw-bug-37`).
