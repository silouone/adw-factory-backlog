---
id: adw-m9-06-lane-wiring
type: feat
status: done
priority: 2
created: 2026-09-12
epic: adw-m9
depends: [adw-m9-04-review-fix-loop, adw-m9-05-verdict-in-pr-body]
attempts: [{"runId":"adw-m9-06-lane-wiring-1789646791624","branch":"adw/adw-m9-06-lane-wiring","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-m9-06-lane-wiring-1789646791624/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/69","provider":"claude","model":"sonnet"}]
---
# Wire review into the three lanes — after green gates, before commit

## Context

`specs/adw-v1.3-review-lane.md` §3.1, R7, R8. The last ticket: everything
before it is off to the side until a lane actually runs it.

**Placement — after green gates, before commit.** Reviewing a red workspace
wastes tokens on findings the repair loop is already fixing; reviewing after
`open-pr` means the human sees the PR before the machine's opinion of it.

    … gates (↻ repair ≤3) → review (↻ review-fix ≤2) → commit → push → open-pr

## Deliverables

`src/pipeline/lanes/{chore,bug,feat}.ts` and the lane tests.

## Requirements

- [ ] All three lanes gain the review stage at the position above. If
      `adw-m8-10` (lane wrapper consolidation) has landed, add it ONCE in the
      shared tail rather than three times — n=3 is exactly the threshold that
      ticket exists for.
- [ ] The `push` node's structural S2.7 guard must still hold: push runs only
      on an all-green gates result patched onto `ctx.data`. Inserting a stage
      between gates and commit must not weaken it — add a test that a blocked
      review cannot reach push.
- [ ] **R8 — per-ticket opt-out.** Ticket frontmatter can skip review, for the
      same reason `caps:` and `model:` exist: a one-line typo fix should not
      pay a review round. Follow the `caps:` precedent — resolved by the
      orchestrator, never by mutating the lane.
- [ ] **Spec D2 is still open — do NOT decide it here.** Wire `chore` with
      review ENABLED, since R8 makes it opt-out-able and the operator can flip
      the default once there is journal data. Note it in the ticket result so
      the choice is visible rather than inherited by accident.
- [ ] R7 — the stage journals `node-start`/`node-end` with the verdict in a
      typed `details` bag, like the gate and agent nodes. "What did the
      reviewer object to, and what happened next" must be answerable from
      artifacts alone (Art. VI).
- [ ] The run-wide minutes cap is now under more pressure (v1.1 Residual Risk
      4: more agent stages pressure the budget). Do NOT raise the default cap
      in this ticket — record the observed cost in the result so it can be
      tuned from data (decision 15 precedent).

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first, per lane: the node sequence has review between gates and
      commit; a blocked review never reaches push; the opt-out skips the stage
      entirely (no agent query made, asserted on the fake query).
- [ ] The engine-flow tests for all three lanes updated.
- [ ] A live worktree run on the SELF target reaching `open-pr` with a review
      section in the PR body. This is the ticket's real bar — the offline tests
      prove wiring, not that a reviewer says anything useful.

## Out of scope

Deciding spec D2 (does chore keep review), D4 (per-role model — see roadmap
2.6), D5 (plan-artifact visibility), D6 (self-review). All four want journal
data from this ticket's runs.
