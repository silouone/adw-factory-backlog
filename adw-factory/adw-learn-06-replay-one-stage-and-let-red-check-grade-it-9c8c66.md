---
id: adw-learn-06-replay-one-stage-and-let-red-check-grade-it-9c8c66
type: feat
status: queued
priority: 2
created: 2026-09-28
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [adw-learn-01-bank-the-native-transcript-42ef45, adw-learn-02-a-run-names-the-factory-that-produced-it-3ceb3e, adw-learn-05-fence-a-held-out-eval-set-0ad553]
attempts: []
---
# Replay one agent stage against a frozen workspace, and let red-check grade it

> The first experiment of the `adw-learn-*` group
> (`ai_docs/2026-09-28-prompt-training-readiness.md` §4). This is the one
> ticket in the group that spends tokens when it is **used**; building it
> does not. It is where the story stops being plumbing.

## Why this stage, why this grader

`build-test-only` is the best-instrumented agent stage in the factory:

| property | measured 2026-09-28 |
|---|---|
| sessions banked | 33 |
| average turns | 31, the shortest of any build-shaped stage |
| grader | `red-check`, deterministic, already classifies `clean-red`, `test-passed`, `broken-test` |
| inputs | ticket body + `plan.md` + rendered `bug-build-test.md`, nothing else crosses the boundary |
| workspace at node-start | `baseline.sha` plus a clean worktree, fully reconstructible |

Because the grader is a production node, a replayed stage is scored by the
factory's own definition of done, not by an LLM judge or a rubric. No
router-agent design can claim that. The same shape then extends to
`build-fix → gates` (30 seeds) and `build → gates` (55 seeds).

## Requirements

- [ ] **R1** A pure `ReplaySpec`: `{ ticketId, target, baseSha, ticketBody,
      plan, templateOverride?: string, label }`. Built from a hold-out entry
      (`eval/holdout.json`) by a loader that reads the ticket at its
      `ticketCommit` from the ledger repo and `plan.md` from the banked run.
      No I/O in the spec itself (Art. IX).
- [ ] **R2** `adw replay --stage build-test-only --holdout <n> [--template
      <path>] [--label <name>]`: for the first `n` bug-lane hold-out entries,
      cut a throwaway worktree at `baseSha`, render `bug-build-test.md` (or
      the override file) with the real `assemblePrompt`, run **one** agent
      node, then run the real `red-check` node, and stop. No commit, no push,
      no PR, no ticket status change, ever. Each replay writes a normal
      `runs/<runId>/` with `run-start.replay: { label, holdoutIndex,
      templateHash }` so `adw-learn-02`'s grouping and `adw-learn-01`'s
      transcripts apply unchanged.
- [ ] **R3** `scripts/replay-report.ts <label>`: per label, the red-check
      classification distribution, turns, context per turn, wall clock and
      tokens per entry, plus a side-by-side of two labels. Reproducible from
      the journals alone.
- [ ] **R4** The replay is refused, before any token, unless every entry it
      would touch passes `scripts/holdout-check.ts`, and unless the fence
      test of `adw-learn-05` R3 recognises the replay tag.
- [ ] **R5** A dry run: `--dry-run` prints each entry's rendered prompt hash,
      base sha and worktree path, and spends nothing. Unlike the CLI's
      current `--dry-run`, it must work.

## The first experiment, once built

1. `adw replay --stage build-test-only --holdout 10 --label control`
2. Change **one** clause in a copy of `bug-build-test.md`.
3. `adw replay --stage build-test-only --holdout 10 --template <copy>
   --label variant-a`
4. `bun scripts/replay-report.ts control variant-a`

Ten entries per label is a coarse grader and about two hours of wall clock.
It is honest, and it exercises the whole harness. Whether to spend the window
on it is the operator's call each time.

## Verify

- [ ] Red first (Art. I). The agent boundary is the injected `AgentQuery`,
      so every test runs on a fake and spends nothing.
- [ ] A replay against a fake agent that writes a failing test → `clean-red`
      in the journal, no commit on the worktree, ticket file untouched.
- [ ] A replay on an entry failing `holdout-check` → refused, exit 2, before
      provision.
- [ ] `--dry-run` spends zero tokens and prints ten prompt hashes.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Any automated mutation of templates, any search over variants, any LLM judge.
Replaying `build-fix` or `build`: same harness, separate tickets once this one
has produced a report the operator has read.
