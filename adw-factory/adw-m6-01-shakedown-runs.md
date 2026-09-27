---
id: adw-m6-01-shakedown-runs
type: chore
status: done
priority: 2
created: 2026-07-14
epic: adw-m6
depends: [adw-m3]
attempts: []
---
# Shakedown runs — real cLens chores

> Refined at pickup 2026-07-15 (plan §8), operator-approved. Original coarse
> scope kept below for provenance.

## Live-backlog survey (2026-07-15, cLens committed HEAD `2b7c2df`)

- Tests green (CLI + 320 web), typecheck green.
- **Lint RED at HEAD**: CLI 2 errors + 82 warnings + 27 infos; web 17 errors
  + 51 warnings + 35 infos. All 19 errors are safe-fixable
  `assist/source/organizeImports`. Warning pool: useTemplate,
  noAccumulatingSpread, noNonNullAssertion, noUnusedImports/Variables,
  useLiteralKeys.
- Spec-time candidate "coverage/ + logs/ gitignore hygiene" is already done
  (both ignored, untracked) — replaced from the live pool per S7.2 ("final
  pick happens at shakedown").
- Dependency patch bumps available: @biomejs/biome 2.5.3→2.5.4 (cli),
  autoprefixer 10.5.2→10.5.3 (web).
- cLens has no `tickets/` dir yet — created as part of this ticket.
- Working copy has 4 uncommitted modified files — E1: factory proceeds
  (worktrees cut from origin/main); only ticket-file edits block dispatch.

## Requirements

**Chore picks (operator-approved), dispatched serially in this order:**

1. `clens-001-lint-errors` — make `bun run lint` exit 0 (fix all 19 errors,
   both packages). **Sequencing constraint: must merge FIRST** — lint is a
   factory gate for this target and it is red at HEAD, so every other run
   fails gates until this lands.
2. `clens-002-cli-lint-warnings` — biome-clean `packages/cli` (82 warnings
   + 27 infos).
3. `clens-003-web-lint-warnings` — biome-clean `packages/web` (51 warnings
   + 35 infos).
4. `clens-004-dep-patch-bumps` — patch-level bumps (biome 2.5.4,
   autoprefixer 10.5.3) + lockfile. Slack ticket: blocked runs elsewhere are
   data, not failures (reopen per protocol and record).

**Factory pre-flight (TDD, red-first, validator-gated) before any live run:**

- [ ] stale-GITHUB_TOKEN fail-fast guard in `src/cli.ts` main() (M2/M3
      watch item; the M2 shakedown script already has it).
- [ ] scoped `allowedTools` incl. Bash on the build/repair agent options
      (operator-approved 2026-07-15; options pinned by exact tests in
      test/pipeline/nodes/agents.test.ts). Unlocks the live TRACEPARENT
      receipt probe.

**Live-run protocol:**

- Copy `runs/m2-scratch/targets/scratch.json`-style config: the clens target
  already exists at `targets/clens.json` — verify, do not duplicate.
- Author the 4 chore tickets in cLens `tickets/` per plan §5 contract,
  commit on cLens main (zero human-written code — ticket authoring and PR
  review only, metric 1).
- Dispatch: `env -u GITHUB_TOKEN -u GH_TOKEN bun src/cli.ts run --target
  clens --ticket <id> --isolation worktree` — explicit operator go-ahead
  per run (AskUserQuestion).
- Operator reviews and merges each PR (factory stays merge-incapable,
  Art. IV).
- Confidence probe on run 1 (operator-approved): verify from the captured
  session that the agent process actually sees TRACEPARENT (the one leg
  unproven live).

## Amendments at run time (operator-approved 2026-07-15)

- **Run 1 (clens-001, `clens-001-lint-errors-1784130060111`): blocked after
  3 repair rounds — recorded as data.** Autopsy: the lint gate (the
  ticket's objective) passed in all four gate rounds; the TEST gate failed
  every round because cLens's `global-read`/`session-registry` tests hit
  the machine's real `~/.clens` registry (35 projects incl. the factory's
  own run dirs) — the two scanning tests take 35s+ against bun's 5s
  timeout (isolated: 9 tests in 515s) and the tests write fixture entries
  into the real registry. A genuine product bug the factory surfaced
  (Story 7 working as intended).
- **clens-005-hermetic-global-tests added to the series, dispatched
  first** (env-overridable `CLENS_GLOBAL_DIR` seam + hermetic tests);
  clens-001 requeues after clens-005 merges.
- **allowedTools verified live in run 1:** the captured session shows the
  agent successfully running `bun run test` / `bun run typecheck` /
  biome — the M3 blocked-Bash turn-burner is gone.
- **Series delegation:** operator delegated requeues + serial dispatches
  of the approved list (clens-001..005) without per-run questions; PR
  review/merge stays with the operator; every outcome reported.
- **Run 2 (clens-005, `clens-005-hermetic-global-tests-1784136713704`):
  blocked by a FACTORY bug — recorded as data.** The agent went green on
  all gates in one repair round (hermeticity seam + it absorbed the
  clens-001 lint errors that were masking the web errors behind the `&&`
  short-circuit), but the commit node's sweep failed: on a target that
  gitignores `.claude/`, a `git add` pathspec naming the ignored path
  hard-fails (git 2.37; `advice.addIgnoredFile=false` does not rescue).
  Fixed red-first in-factory: two-step sweep (`git add -A -- .` +
  `git reset -q -- .claude`) in the commit and ci-commit nodes; porcelain
  dirty checks keep the exclude form. clens-005 requeued after the fix.

- **TRACEPARENT receipt proven live (probe run m3s-002-1784146495335,
  scratch repo, 2026-07-15):** the agent-written TRACE.md is byte-exact
  `00-<traceId>-<build spanId>-01` for that run's exported build span, and
  traceId == sha256(runId)[0..32). Scratch PR #5 carries the artifact.
- **Turn-cap fix + caps 480/400 verified live (runs 5–7):** clens-002
  green at num_turns 261 (would have breached the old 200 boundary cap);
  clens-003 one-shot at 139; clens-004 at 58. Spec Story 2 turn semantics
  amended (operator-approved).

## Verify

≥ 3 factory PRs merged into cLens; each run's artifacts complete (S7.1):
journal + spans + tagged captured session per the m3-05 runbook reader
rules. Factory suite `bun run lint && bunx tsc --noEmit && bun test` green
after the pre-flight changes.

---

## Original coarse scope (pre-refinement)

Pick ≥ 3 genuine chores from cLens's live backlog (candidate pool at spec
time: `coverage/`/`logs/` gitignore hygiene, outstanding lint/typecheck
warnings, dependency bumps — S7.2). Write each as a `tickets/<id>.md` chore
in the cLens repo per the plan §5 contract, dispatch through the full lane
under `--isolation worktree` (plan §8 as reordered), review and merge as
operator. Zero human-written code — only ticket authoring and PR review
(metric 1). Blocked runs are data, not failures: reopen per protocol and
record.
