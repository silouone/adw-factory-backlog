---
id: adw-m9-05-verdict-in-pr-body
type: feat
status: done
priority: 2
created: 2026-09-12
epic: adw-m9
depends: [adw-m9-03-two-axis-stages]
attempts: [{"runId":"adw-m9-05-verdict-in-pr-body-1789372484440","branch":"adw/adw-m9-05-verdict-in-pr-body","workspace":"/Users/silouane/adw-factory/runs/adw-m9-05-verdict-in-pr-body-1789372484440/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/37","provider":"claude","model":"sonnet"}]
---
# The verdict reaches the human — findings in the PR body

## Context

`specs/adw-v1.3-review-lane.md` R6. This is the requirement that converts
review from a cost into a briefing, and it is the one that makes the epic's
kill-metric measurable: if operator review minutes per PR do not fall, the
whole amendment added cost and changed nothing.

`tickets/BACKLOG.md` #5 named the problem — every current §6 metric is UPSTREAM
of review, and nothing measures reviewer minutes. Art. IV puts the human exactly
there, and the PR body is the only surface that reaches them.

Independent of adw-m9-04: a reviewer that only reports is already worth
shipping, so this must not depend on the fix loop.

## Deliverables

`prompts/pr-body.md` gains an optional review section; `open-pr` threads it.

## Requirements

- [ ] `prompts/pr-body.md` gains an OPTIONAL review placeholder. Optional
      because `chore` may skip review (spec D2) and a ticket may opt out (R8) —
      a required placeholder would break those runs (the `{{plan}}` precedent,
      which is optional for exactly this reason).
- [ ] Findings are rendered UNDER THEIR AXIS — `## Standards` and `## Spec`,
      separately. Never merged, never re-ranked across axes (R2). The
      separation must survive into the body or it was pointless.
- [ ] Each finding shows severity and its citation (the standard, or the ticket
      line) so the human can judge it without re-deriving it.
- [ ] Findings FIXED by a review↻fix round are shown as fixed, not dropped.
      What the reviewer caught and the builder then fixed is the highest-value
      line in the whole body — it is the evidence the loop works.
- [ ] A reviewed run with zero findings says so explicitly. Silence is
      ambiguous between "clean" and "review did not run".
- [ ] `open-pr` reads the verdicts from `ctx.data` — it already reads
      `agentSummary` and the gate results, so this is one more read, not new
      plumbing (Art. VIII).

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first: a body rendered with findings on both axes keeps them
      separate; a body with no verdict at all renders unchanged (the opt-out
      path); a zero-findings verdict renders the explicit clean line.
- [ ] The `open-pr` snapshot test is updated deliberately, not blessed blindly.

## Out of scope

The fix loop (adw-m9-04). Measuring reviewer minutes — that is BACKLOG #5 /
roadmap 2.7, and needs its own ticket once this has shipped and there is
something to measure.
