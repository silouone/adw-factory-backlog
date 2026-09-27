---
id: adw-m9-03-two-axis-stages
type: feat
status: done
priority: 1
created: 2026-09-12
epic: adw-m9
depends: [adw-m9-02-review-node]
attempts: [{"runId":"adw-m9-03-two-axis-stages-1789367511540","branch":"adw/adw-m9-03-two-axis-stages","workspace":"/Users/silouane/adw-factory/runs/adw-m9-03-two-axis-stages-1789367511540/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Two axes — Standards and Spec, parallel, never merged

## Context

`specs/adw-v1.3-review-lane.md` R2. Ported from the operator's `code-review`
skill, which has gated seven milestone exits by hand.

The separation is the whole point: a change can pass one axis and fail the
other. Code that follows every standard but implements the wrong thing is
Standards-pass / Spec-fail. Reporting them together lets one mask the other, so
they are never merged and never re-ranked against each other.

The factory has an advantage the skill lacks: **the spec source is unambiguous.**
The skill must hunt for a PRD; here the ticket IS the contract.

## Deliverables

`prompts/review-standards.md`, `prompts/review-spec.md`, and the two node
instances wired as a parallel pair.

## Requirements

- [ ] Two `review` node instances (adw-m9-02), one per axis, each with its own
      prompt and its own `ctx.data` key.
- [ ] They run in PARALLEL — neither axis' findings may enter the other's
      context. Sequential execution would leak axis 1's framing into axis 2.
- [ ] `review-standards.md` asks for: places the diff breaches a DOCUMENTED
      repo standard (citing file + rule), and baseline code smells. It must
      state that a documented repo standard OVERRIDES the baseline, that
      baseline smells are ALWAYS judgement calls, and that anything tooling
      already enforces is skipped — duplicating `gates` wastes the round and
      Art. III keeps deterministic checks deterministic.
- [ ] `review-spec.md` asks the three spec questions, mapped to adw-m9-01's
      `SpecFinding` discriminants: `missing`, `unasked`, `wrong`. Each finding
      must quote the ticket line it is measured against.
- [ ] Both prompts thread the existing `{{ticketBody}}` / `{{conventions}}` /
      `{{contextPointers}}` placeholders via `assemblePrompt`, and both must
      pass `test/pipeline/templates.test.ts`' committed-template placeholder
      scan (N5) — no typo'd placeholder reaches a live lane.
- [ ] Both prompts instruct the agent to emit ONLY the verdict shape
      adw-m9-01's `parseVerdict` accepts.
- [ ] Neither prompt inherits the no-weakening rule's phrasing about editing
      tests — a reviewer cannot edit anything (R4). Do not copy it in.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first: both axes produce independent verdicts from a fake
      query; a test asserts the standards verdict is NOT present in the spec
      node's prompt and vice versa.
- [ ] The committed-template placeholder scan covers both new prompts.

## Out of scope

Routing (adw-m9-04), the PR body (adw-m9-05), lane wiring (adw-m9-06).
Whether the reviewer sees the plan artifact — spec D5, still open; do not wire
it in this ticket.

---

## Resolution — 2026-09-14

Merged as PR #35. The run (`adw-m9-03-two-axis-stages-1789367511540`) completed
its `test` stage and was then stopped by the no-progress detector
(`adw-auto-02`) — its FIRST live trip, and correct by its own contract: the
agent ran `true` three times consecutively with no distinct successful action
between. The trip aborted the stream before the result message, so the node
failed and the run blocked with the work already done and green.

Salvaged by hand and verified on its unchanged base (`b36d82b`): lint clean,
tsc clean, `bun test` 1486 pass / 4 skip / 0 fail, 0 docker skips.

`status: blocked` was therefore the run's accurate record of how it ENDED, and
stale as a record of the WORK. Closed here.
