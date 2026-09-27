---
id: adw-review-05-when-a-ticket-may-skip-the-reviewer
type: chore
status: done
priority: 2
created: 2026-09-18
review: false
caps: {minutes: 30, turns: 150}
depends: []
attempts: [{"runId":"adw-review-05-when-a-ticket-may-skip-the-reviewer-1789769657518","branch":"adw/adw-review-05-when-a-ticket-may-skip-the-reviewer","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-review-05-when-a-ticket-may-skip-the-reviewer-1789769657518/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/83","provider":"claude","model":"sonnet"}]
---
# `review: false` exists and nothing says when to use it

> **Spec authority:** `specs/adw-v1.13-run-economics.md` §3 **S1**, §5 **D1**.

## Why

Review stages are **284.2m — 19.6% of model-stage time** across the 25 banked
runs over 40 minutes, and the review loop is the single largest cause of
`blocked`. The opt-out is already wired end-to-end and journalled as an
explicit opt-out, never a silent skip.

What is missing is **policy**. Without a written rule the opt-out becomes either
a habit the operator drifts into, or a field nobody remembers exists. Both are
worse than a stated criterion.

**This ticket ships no code.** The field is parsed, validated, threaded to the
lane and consumed where the review pair is spliced. Confirm that end-to-end
path still holds, then write the rule down.

## Requirements

- [ ] **R1 — the criterion, in `tickets/README.md`.** A ticket qualifies as
      hard-gated — and may carry `review: false` — when **every** acceptance
      criterion is expressed as a gate command or a machine-checkable assertion
      in its Verify block. Judgement-shaped work (naming, API shape,
      architecture, anything where "is this the right design" is the question)
      **does not qualify**.
- [ ] **R2 — worked examples, both directions.** Cite a ticket in this repo that
      qualifies and one that does not, and say why in one line each.
- [ ] **R3 — say what it costs.** The operator is trading a second opinion for
      ~20% of model time and a large share of blocked risk. State that
      plainly so the choice is informed rather than reflexive.
- [ ] **R4 — confirm the wiring, do not assume it.** Trace `review: false` from
      frontmatter through to the lane skipping the review pair, and confirm the
      journal records the opt-out. Paste the journal line as evidence.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green (no code changed).
- [ ] `tickets/README.md` carries R1's criterion, R2's two examples and R3's
      cost statement.
- [ ] The journal line proving the opt-out is recorded is pasted into this
      ticket.

## Out of scope

- Any code change. If the wiring turns out to be broken, **stop and raise a
  bug ticket** — do not fix it here.
- Applying `review: false` to existing tickets. This writes the rule; the
  operator applies it.
