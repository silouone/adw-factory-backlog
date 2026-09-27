---
id: adw-m2-01-push-node
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m2
depends: [adw-m1]
attempts: []
---
# Push node

## Context

"The factory shall never push work for which any local gate is failing"
(S2.7) — the guard is the point of this node, not the push. Nothing is ever
pushed to a protected/default branch (Art. IV, N6).

## Deliverables

- `test/pipeline/nodes/push.test.ts` (first, red — fake Workspace exec)
- `src/pipeline/nodes/push.ts`

## Requirements

- [x] Precondition enforced structurally: push runs only when ctx carries an
      all-green gates result; anything else → `fail` with a descriptive
      error — test the guard explicitly, not just the happy path (S2.7)
- [x] `git push -u origin <attemptBranch>` via `workspace.exec`; the ref
      pushed is always the attempt branch, never `target.base` (N6, Art. IV)
- [x] Push failure (network/auth, faked): deterministic bounded retries
      (3 attempts), then `fail(blocked)` with the local branch name recorded
      for the journal — branch left intact (E6)
- [x] The lane gains a DETERMINISTIC COMMIT NODE before push, and push
      additionally refuses a dirty workspace as a belt-and-braces guard.
      Root cause from run obs-001-1784056112585 (2026-07-14, transcript
      forensics, corrected): NOT a custom hook — Claude Code's built-in
      permission layer. Headless SDK sessions run `permissionMode:
      "acceptEdits"`, which auto-accepts file edits only; Bash commands
      mutating `.git` (add/commit/update-index) require interactive
      approval, and headless auto-denies. The build agent tried 10+ commit
      variants, was systematically denied, and correctly surfaced the
      blocker in its final report (which the CLI then discarded — see the
      companion finding). Fix: factory-owned deterministic commit node
      (works in any environment, deterministic message, Art. III);
      documented alternative rejected: `allowedTools: ["Bash(git add:*)",
      "Bash(git commit:*)"]` in SDK options. Also: the CLI must print the
      agent's final summary on run completion — the surfaced blocker was
      invisible to the operator.

## Build protocol (Art. I)

1. Fake exec scripted to succeed / fail N times / always fail. Tests: guard;
   ref correctness; retry count; blocked outcome carries branch name.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

PR creation (adw-m2-02), CI (adw-m2-04).
