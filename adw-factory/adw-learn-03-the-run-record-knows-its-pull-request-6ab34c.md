---
id: adw-learn-03-the-run-record-knows-its-pull-request-6ab34c
type: feat
status: blocked
priority: 2
created: 2026-09-28
review: false
caps: {minutes: 180, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-learn-03-the-run-record-knows-its-pull-request-6ab34c-1790589231282","branch":"adw/adw-learn-03-the-run-record-knows-its-pull-request-6ab34c","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-learn-03-the-run-record-knows-its-pull-request-6ab34c-1790589231282/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The run record knows its pull request, and learns what the human did to it

> `adw-learn-*` group (`ai_docs/2026-09-28-prompt-training-readiness.md` §3.3).
> Zero-token plumbing. `review: false`: hard-gated.

## The defect this closes

`open-pr`'s `node-end` details are `null` on all 59 green runs in `runs/`.
The PR URL reaches the ticket's `attempts[]` entry and the CLI's stdout, and
nowhere else. `sync-pr-state` moves the ticket to `done` or `rejected` when the
human merges or closes, and records nothing about the run that produced the
PR.

The human review is the only human-quality label in the whole system:
merged as-is, merged after fixup commits, closed, how many review comments,
how long it sat. The run record cannot reach any of it. 135 `adw/` PRs are
merged on adw-factory and 9 on clens; a training corpus that cannot join a
run to its PR's fate has no top-level label.

## Requirements

- [ ] **R1** `open-pr`'s `node-end` details carry
      `{ kind: "pr", url, number, repo }`, parsed from `gh pr create`'s
      output by a pure function with a banked real payload as its fixture
      (the rule `adw-gates-07` R4 established: never a hand-written fixture
      for a parser of external output).
- [ ] **R2** A new journal event `pr-state`, appended by `sync-pr-state` to
      the **producing run's** journal when it observes a terminal PR state:
      `{ type: "pr-state", number, state: "merged" | "closed", at,
      reviewComments, reviews, fixupCommits }`. `fixupCommits` is the count of
      commits on the PR branch after the factory's own commit; `reviews` is
      the count of formal review submissions. The producing run is found via
      the ticket's `attempts[].runId` whose `pr` matches; when no run
      directory exists any more (pruned), the event is skipped with a notice,
      never an error.
- [ ] **R3** The journal is append-only and a run's journal may be days old
      when `pr-state` lands. Readers that assume `run-end` is the last line
      (`just outcome`, the web `RunView`) must tolerate a trailing `pr-state`;
      `capture` downgrade records after `run-end` are the precedent.
- [ ] **R4** `scripts/run-metrics.ts` gains a **merged** column beside green,
      recomputed from `pr-state`, and reports the green-but-not-merged and
      the merged-with-fixups counts by lane.
- [ ] **R5** The web card shows the PR number and its terminal state when
      known. No other UI change.

## Verify

- [ ] Red first (Art. I).
- [ ] A fake `gh pr create` output → `{ url, number, repo }` on `open-pr`'s
      node-end; malformed output is a descriptive `fail`, not a crash.
- [ ] A fake `gh pr view` reporting merged with two review comments and one
      fixup commit → one `pr-state` line appended to the producing run's
      journal, and `just outcome` still reads `green`.
- [ ] A ticket whose producing run directory is gone → notice, no event, no
      error.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Reading review comment **bodies** into the run record; they already reach the
ticket on rejection (`appendReviewFeedback`). Polling GitHub more often than
`sync-pr-state` already does: `adw-pr-02` owns the one-read-per-target TTL.
