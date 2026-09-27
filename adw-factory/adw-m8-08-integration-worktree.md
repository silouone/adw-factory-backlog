---
id: adw-m8-08-integration-worktree
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m8-06-feat-lane, adw-m8-07-bug-lane]
attempts: []
---
# Stage 1 live bar — a real `bug` and a `feat` merged on the worktree kind

> Source of truth: `specs/adw-v1.1-lanes.md` (Acceptance §Stage 1). Read
> constitution + specs first. **Operator-gated** (real tokens + real cLens PRs).

## Why a live bar (not offline-green)

Offline-green masked real-wiring bugs **twice** in M7. The Planner-led feature
pipeline and the two-phase red-check must be proven end-to-end on a real target,
where it is cheapest and most observable: the **worktree** kind.

## Requirements

- [ ] A real cLens **`bug`** ticket travels Graph C on **worktree** to a
      **merged** PR: base-green passes → plan → phase-1 writes a reproducing test
      → red-check confirms **clean red** (test red, lint+typecheck green) →
      phase-2 (resumed) fixes → gates green → PR → operator merges. The merged PR
      carries the regression test.
- [ ] A real cLens **`feat`** ticket travels Graph B on **worktree** to a
      **merged** PR: plan → build (tests-first) → test (coverage) → gates green →
      PR. The merged PR carries planned, tested feature work.
- [ ] Both runs **replayable** from journal + spans + captured sessions (Art. VI);
      the bug run's journal shows base-green and clean-red verdicts with reasons;
      the feat run's journal shows the plan artifact feeding the build.
- [ ] `chore` on worktree still works (spot regression check).

## Build protocol

1. Author/pick a genuinely mechanical cLens `bug` (reproducible defect) and a
   mechanically-shaped `feat` (spec pointer in the body).
2. Dispatch each on worktree; observe the stages + red-check live; record run ids
   / PR numbers here.
3. Operator reviews and merges each PR.

## Live-bar preconditions + readiness (prep — 2026-07-23, offline 01–07 done)

The offline suite (789 green) structurally CANNOT check these two — they bite
only on a live dispatch, so verify them before/while dispatching:

- **base-green must be green at cut time.** `base-green-check` runs the target's
  FULL gate suite (`lint`+`typecheck`+`test`) on the untouched worktree and
  BLOCKS on any red before the plan agent spends a token (decision 9). So confirm
  cLens `main` is fully gate-green when the run cuts its worktree. Watch: cLens
  `test` is git-heavy and can flake under load (5s timeouts, per M3/M7 live
  notes) — a flaky `test` gate on `main` will flap base-green. Pre-run
  `bun run lint && bun run typecheck && bun run test` in the cLens repo on `main`
  to confirm green (and that the primary working copy is clean —
  `git pull --rebase --autostash`, cLens dev leaves uncommitted files).
- **the reproducing test must be a CLEAN red** (bug only): the `test` gate fails
  while `lint`+`typecheck` stay green. If the phase-1 test also breaks lint/type
  (broken-test) or doesn't reproduce (test-passed), red-check drives the bounded
  revise loop (cap 2) then BLOCKS. So an early block after ≤2 revises reads as
  "the agent needed its revise tries," NOT "the lane is broken." Pick a defect
  whose regression test is a clean red (a behavioral bug, not a type/lint issue).

**Readiness:** `targets/clens.json` is wired (repo
`/Users/silouane/agent-observability-project`, gates lint/typecheck/test → the
`test`-gate convention holds, so the bug fail-fast ACCEPTS this target). Dispatch
(operator-gated, real tokens; the operator merges — Art. IV):
`env -u GITHUB_TOKEN -u GH_TOKEN bun src/cli.ts run --target clens --isolation worktree --ticket <id>`.
Still to author (needs a cLens exploration pass): a genuinely mechanical `bug`
(reproducible behavioral defect + a clean-red regression test) and a
mechanically-shaped `feat` (spec pointer in the body).

## Runs (Stage 1 — worktree, 2026-07-23)

Both new lanes traveled to a green PR on a real cLens worktree — **awaiting the
operator's merge** (Art. IV; the factory is merge-incapable):

- **`bug` (Graph C) — clens-009-edits-highlight-mismatch → PR silouone/clens#18.**
  Run `clens-009-edits-highlight-mismatch-1784800379163`. Full red-first flow
  ran: base-green-check `[lint:pass, typecheck:pass, test:pass]` → plan →
  build-test-only → **red-check `[lint:pass, typecheck:pass, test:fail]` =
  clean-red** (the machine watched the reproducing test go red before the fix,
  first pass — no revise needed) → build-fix (RESUME) → gates green (1866 pass,
  incl. the 4 now-green regression tests). Journal replayable (Art. VI); the
  clean-red verdict + its gate-result array are recorded. Agent fixed both
  `groupFilesByDirectory` and its twin `groupFilesByAgent` by identity lookup.
- **`feat` (Graph B) — clens-010-import-codex-autodiscover → PR silouone/clens#19.**
  Run `clens-010-import-codex-autodiscover-1784802254898`. plan → build
  (tests-first) → test (coverage) → gates green (1885 CLI + 320 web pass). The
  **test agent earned its keep** — hardened the untested `USERPROFILE`/empty-
  `CODEX_HOME` fallback branches the build missed (and dropped a hermeticity-
  breaking test on review). Plan→build `{{plan}}` threading proven live.

Live finding folded (not a lane bug): the first feat dispatch BLOCKED cleanly
because cLens `origin/main` was 8 commits behind local `main` — the codex
importer (clens-007) it extends was unpushed, and worktrees cut from
`origin/base` by design (plan §5). The feat lane executed every stage; the agent
correctly refused to invent the missing foundation (amendment rule). Resolved:
operator-authorized fast-forward push of cLens `main` to origin (d0d8241 →
ab1b7c0) + repointed the ticket off the gitignored `specs/` doc to the in-repo
importer, then re-dispatched → green. Lesson: a target's `origin/base` must carry
a bug/feat's prerequisites; a feat extending unpushed local work blocks by design.

## Verify

Two merged cLens PRs (one `bug`, one `feat`) on worktree; each replayable; the
bug PR's test provably went red before the fix. Record run ids + PR numbers here.
Status: both PRs OPEN + green (#18 bug, #19 feat) — **operator merge pending** to
close Stage 1.

## Out of scope

The E2B moat (m8-09) — the mandatory done bar.
