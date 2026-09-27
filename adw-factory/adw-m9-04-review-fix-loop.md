---
id: adw-m9-04-review-fix-loop
type: feat
status: done
priority: 2
created: 2026-09-12
epic: adw-m9
depends: [adw-m9-03-two-axis-stages, adw-bug-07-plan-and-build-must-write-their-artifacts]
attempts: [{"runId":"adw-m9-04-review-fix-loop-1789402361952","branch":"adw/adw-m9-04-review-fix-loop","workspace":"/Users/silouane/adw-factory/runs/adw-m9-04-review-fix-loop-1789402361952/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-m9-04-review-fix-loop-1789422475817","branch":"adw/adw-m9-04-review-fix-loop-2","workspace":"/Users/silouane/adw-factory/runs/adw-m9-04-review-fix-loop-1789422475817/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-m9-04-review-fix-loop-1789485382215","branch":"adw/adw-m9-04-review-fix-loop-4","workspace":"/Users/silouane/adw-factory/runs/adw-m9-04-review-fix-loop-1789485382215/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-m9-04-review-fix-loop-1789607658815","branch":"adw/adw-m9-04-review-fix-loop","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-m9-04-review-fix-loop-1789607658815/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-m9-04-review-fix-loop-1789637947039","branch":"adw/adw-m9-04-review-fix-loop-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-m9-04-review-fix-loop-1789637947039/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/67","provider":"claude","model":"sonnet"}]
---
# Review sends work back — to a FRESH builder, bounded

## Context

`specs/adw-v1.3-review-lane.md` R5, with D1 and D7 decided 2026-09-12.

A reviewer that only reports is a better PR body. A reviewer that can return
work to the builder is the actual capability. Art. V binds it: every loop has a
hard ceiling declared in code.

**D1 (decided) — the floor.** Work is sent back for Standards-axis
`severity: "hard"` and Spec-axis `kind: "missing"`. Judgement calls, baseline
smells and scope-creep notes do NOT route; they ride into the PR body
(adw-m9-05). `isBlocking` from adw-m9-01 already encodes this — USE IT, do not
re-implement the predicate here.

**D7 (decided) — a FRESH build, not a resumed session.** The agent that wrote
the code is the worst candidate to judge whether the criticism lands; resume
invites it to re-litigate its own choices rather than act on them, and this is
the one node whose entire purpose is to act on someone else's judgement.
Mechanically this makes the fix stage an instance of the EXISTING `build` node
(like `plan`/`test`), so no fourth agent node type appears.

## Deliverables

`prompts/review-fix.md`, the routing wiring, and tests.

## Requirements

- [ ] After both axes complete, the union of their findings is filtered by
      `isBlocking`. Non-empty → route to the fix stage; empty → advance.
- [ ] The fix stage is a `build` node instance whose prompt carries the
      BLOCKING findings and the diff — and NOT the prior agent's session or
      reasoning (D7).
- [ ] `prompts/review-fix.md` instructs: address each finding or state plainly
      why it is wrong. It must carry the no-weakening rule (commit `b57bca0`) —
      this is a fix-under-pressure stage, exactly where weakening a test is the
      cheap path to silence.
- [ ] A hard ceiling in code on review↻fix rounds. Use the engine's existing
      per-node `retry.maxRounds` override (adw-m8-02) rather than new loop
      machinery (Art. VIII). Propose 2, matching `bug`'s revise cap; the lane
      declares it, the ticket's `caps:` may override.
- [ ] Breaching the ceiling: stop, mark blocked, hand over the transcript
      (Art. V). It must NOT fall through to commit with blocking findings
      outstanding — that would make the loop theatre.
- [ ] After a fix round, gates re-run before review re-runs. A fix can break
      the build, and reviewing a red workspace wastes a round.
- [ ] Each round journals a `round` event naming the node and the round (the
      existing `red-check` / `repair` precedent), so the loop is legible from
      artifacts alone (Art. VI).

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first, fake query: a blocking finding routes to the fix stage;
      a judgement-call-only verdict does NOT route and advances; the ceiling
      blocks after exactly N rounds naming the exhausted ceiling; a fix round
      that breaks gates is caught before review re-runs.
- [ ] A test proving the fix stage's prompt carries NO resumed session id (D7).

## Out of scope

The PR body (adw-m9-05). Lane wiring (adw-m9-06). Tuning the ceiling from
journal data — that needs runs first (decision 15 precedent).
