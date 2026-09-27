---
id: adw-review-06-a-missing-requirement-opens-a-round
type: bug
status: done
priority: 1
created: 2026-09-27
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# A missing requirement ships green: `missing`/`hard` never opens a review-fix round

> **Evidence, 2026-09-27, run `adw-bug-21-a-blocked-run-hides-why-it-blocked-1790463578740`.**
> `review-standards` returned 0 findings. `review-spec` returned 5 findings, all
> `{kind:"missing", severity:"hard"}`: R2, R3 and R4 were not implemented at
> all. `review-spec` ended `next`, and the run went commit → push → open-pr →
> **green** (PR #112). Across all banked journals, the same shape shipped green
> in `adw-cost-02-turn-economy-1789996101755` (#106, merged) and
> `sabado-14-nothing-red-or-drifted-lives-on-main-1789859138376`.

## Root cause

`src/pipeline/review/fix-loop.ts` (`makeReviewGateLoop`, wrapped `spec.run`)
opens a round only when `unionBlockingFindings(...)` is non-empty.
`verdict.ts` `isBlocking` never counts a `spec`/`missing` finding. That was
correct under `specs/adw-v1.12-review-routing.md` §3 as written, which assumed
"missing" meant "a test is absent". The reviewer prompt defines it as a
requirement absent or only partly implemented.

## Spec

`specs/adw-v1.12-review-routing.md` **§3.1** (amendment 2026-09-27, operator
decision): a `missing`/`hard` finding opens **one** review-fix round when no
round has run yet; it still never blocks.

## Requirements

- [x] **R1 — red first.** In `test/pipeline/review/fix-loop.test.ts`:
      standards has no findings, spec returns one `{kind:"missing",
      severity:"hard"}`. On the first round, the wrapped `spec.run` returns
      `kind:"retry"` with that finding in `payload`. This fails on `main`.
- [x] **R2 — only one round.** On a later round (a review-fix already ran),
      the same verdict returns `next`: it never blocks and never exhausts the
      loop into `blocked`. Test it.
- [x] **R3 — unchanged rows.** `missing`/`judgement` alone still returns
      `next`; `unasked` alone still returns `next`; blocking findings behave
      exactly as today (existing tests stay green, unedited).
- [x] **R4 — annotated.** A `missing`/`hard` that survives its round appears
      in the PR body's un-addressed findings (§4). Assert through the
      existing open-pr rendering test path.
- [x] **R5 — how the round is known.** Derive "a round already ran" from
      state the engine already keeps (the retry round counter or
      `ctx.data`). Do not add a second counter. Name the source in a comment.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`.

## Blocked by

None.
