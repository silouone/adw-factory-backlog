---
id: adw-review-04-a-non-converged-review-opens-a-pr
type: feat
status: done
priority: 1
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: [adw-review-02-unresolved-findings-ride-into-the-pr]
attempts: [{"runId":"adw-review-04-a-non-converged-review-opens-a-pr-1789807666289","branch":"adw/adw-review-04-a-non-converged-review-opens-a-pr","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-review-04-a-non-converged-review-opens-a-pr-1789807666289/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/88","provider":"claude","model":"sonnet"}]
---
# 996 minutes of gate-green work was thrown away because a reviewer would not converge

> **Spec authority:** `specs/adw-v1.13-run-economics.md` §3 **S4**, §5 **D4**.
> Amends `specs/adw-v1.md` §5 (the `green | blocked` terminal contract).

## Evidence

Of the 25 banked runs over 40 minutes, **13 ended `blocked` — 996 minutes, 56%
of long-run wall clock, producing nothing.** Almost all died in the review loop,
and **in every one of them `gates` had already gone green.**

```
adw-fe-15  118.7m blocked  gates=[next]  review-spec exhausted 2 rounds
adw-pr-01   86.7m blocked  gates=[next]  review-spec exhausted 2 rounds
adw-fe-22   71.6m blocked  gates=[next]  review-fix broke the test gate
```

`review-fix` costs a mean **15.6m per round**. `adw-bug-13` fixed Defect B
(one flake was terminal); **Defect A — rounds are independent redraws, not
iterations — remains open.** This ticket does not make the loop converge. It
makes non-convergence survivable.

When the review loop exhausts, gates are green **by definition**. The work is
shippable and the findings are advisory. Ship it, and put the findings where
Article IV already puts a human: the pull request.

## Requirements

- [ ] **R1 — exhaustion returns `next`, not `fail`.** The review↻fix loop's
      round-exhaustion path currently calls `fail()`, which the engine turns
      into `blocked`. It instead returns `next`, patching `ctx.data` with the
      unresolved findings and a flag marking the run review-exhausted. The lane
      proceeds to its normal `commit` → `push` → `open-pr` tail.
- [ ] **R2 — render nothing yourself.** `adw-review-02-unresolved-findings-ride-into-the-pr`
      builds the unresolved-findings renderer; this ticket **depends on it** and
      must not duplicate it. Your job is to make an exhausted run *reach*
      `open-pr` carrying those findings in `ctx.data`. If the renderer is not
      there yet, stop — the dependency is unmet.
- [ ] **R3 — the PR says so plainly.** The body states that the review loop
      exhausted its rounds and that the findings below are unresolved. An
      annotated PR must never read as a clean one.
- [ ] **R4 — exactly one terminal path changes.** Gates-repair exhaustion, CI-
      repair exhaustion, watchdog trips, hard stops, and turn/wall-clock
      ceilings **all keep their current behaviour**. A gate failure still
      blocks — the code is genuinely broken. Add a test pinning each of these,
      so this ticket cannot widen later by accident.
- [ ] **R5 — the journal distinguishes them.** A reader must be able to tell a
      clean green from a review-exhausted green from the journal alone, without
      reading the PR.

## Order of work — Article I

1. Write the fix-loop test: given a verdict that never converges, the loop
   returns `next` carrying unresolved findings rather than `fail`. Confirm
   **red**. Prior art in the same file: `adw-bug-13` added a reproducing
   exhaustion test.
2. Write the R4 tests pinning the unchanged paths. They should be **green
   already** — they are a regression fence, not a red test.
3. Implement, then update the `open-pr` snapshot.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] Red output for the fix-loop test pasted into this ticket (Art. I).
- [ ] The `open-pr` snapshot shows the unresolved-findings section and the
      exhaustion statement.
- [ ] A test proves a **gates-repair** exhaustion still ends `blocked`.
- [ ] A test proves a clean green PR body is **unchanged** — no stray section
      when there is nothing unresolved.
- [ ] `specs/adw-v1.md` §5 amended: `green` means "the PR is open", which may
      include unresolved advisory findings. It does not mean "a reviewer
      approved it".

## Out of scope

- **Making the review loop converge** (Defect A of `adw-bug-13`). Separate
  ticket. This one is about what happens when it does not.
- Changing how many rounds the loop gets.
- Any other exhaustion path (R4 pins them).
