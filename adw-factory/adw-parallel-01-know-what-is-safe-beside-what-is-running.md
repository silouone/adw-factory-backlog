---
id: adw-parallel-01-know-what-is-safe-beside-what-is-running
type: feat
status: done
priority: 2
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-parallel-01-know-what-is-safe-beside-what-is-running-1789407493352","branch":"adw/adw-parallel-01-know-what-is-safe-beside-what-is-running","workspace":"/Users/silouane/adw-factory/runs/adw-parallel-01-know-what-is-safe-beside-what-is-running-1789407493352/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/44","provider":"claude","model":"sonnet"}]
---
# `depends:` encodes ordering, not file overlap — so `just next` cannot say what is safe to pair

> Minted 2026-09-14 after four hand-rebases and one abandoned PR in a day.

## Evidence

Five parallel dispatches on 2026-09-14. What happened to each on merge:

| PR | outcome |
|---|---|
| #13 `fe-01` | rebased clean |
| #14 `auto-02` | 1 conflict — both sides appended a "Part C" test section |
| #26 `tools-01` | 2 conflicts, plus `makeRepairNode` had gained a third argument |
| #31 `gates-02` | **silently dropped 5 tests** (below) |
| #30 `m8-10` | **abandoned** — a lane refactor across three landed changes |

And the pairs that merged with **zero** conflicts — `fe-05 ∥ fe-06`,
`fe-02 ∥ fe-07` — are exactly the ones that respected the roadmap's column
rule: left column = the factory chain (`build.ts`, `journal.ts`, `engine.ts`),
right column = new web modules, no file overlap.

**So the rule works. It is just written in a doc
(`ai_docs/2026-09-12-build-roadmap-parallelism.md`) that goes stale, and
nothing in the CLI knows it.** `just next` reports "deps met", and `depends:`
is about ORDERING — `adw-fe-06-gantt` genuinely needs `fe-05` to have happened.
It says nothing about whether two tickets will collide in the same file.

Every conflict above came from pairing on `just next` + deps without checking
files.

## The dangerous one, and why this is not just about convenience

Rebasing `#31` conflicted in `test/pipeline/lanes/bug.test.ts`. Git split the
hunks MID-TEST, so neither side concatenated into valid TypeScript. Taking one
side wholesale rebased **cleanly, with no error and a green build** — and
dropped all five of `adw-resilience-01`'s bug-lane tests. Its SOURCE had merged
fine; only the test file collided, so nothing failed.

It was caught by grepping for the ticket id. **Nothing in the toolchain would
have caught it.** A conflict resolution that silently removes tests is worse
than a conflict that fails loudly.

## Requirements

- [ ] Establish, from data rather than from the roadmap doc, what each open
      ticket is likely to touch. Merged tickets have their PR's real file list;
      unmerged ones have only their body. Say plainly which signal is used and
      how wrong it can be — **a confident wrong answer here is worse than no
      answer**, because it would license a bad pairing.
- [ ] `just next` (or a sibling) can answer "what is safe to start beside what
      is running right now", not only "whose deps are met".
- [ ] The signal must degrade honestly: a ticket whose surface cannot be
      predicted is reported as UNKNOWN, never as safe.
- [ ] Do **not** block a dispatch on it. The factory refuses loudly at real
      preconditions (a bad base ref, a missing test gate); predicted file
      overlap is advice, not a precondition, and the operator stays the
      scheduler (Art. IV).

## Verify

- With runs in flight, the tool names at least one genuinely safe ticket and
  correctly flags one that would collide.
- A ticket with no predictable surface reports UNKNOWN rather than safe.
- `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Automatic rebasing, and detecting tests lost in a conflict resolution. The
second is a real gap this ticket's evidence exposes, but it is a different
mechanism (comparing the test inventory across a rebase) and wants its own
ticket — named here so it is not forgotten.
