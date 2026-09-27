---
id: adw-selfhost-lint-gate
type: bug
status: done
priority: 1
created: 2026-09-11
depends: []
attempts: []
---
# The lint gate is vacuously red for every self-target worktree run

## Symptom

Found 2026-09-11 by the first self-target feat run
(`adw-m5-06-remote-capture-parity-1789078287643`). Inside the run's worktree:

    $ bun run lint          # = biome check .
    × No files were processed in the specified paths.
    i These paths were provided but ignored:
    - .
    error: script "lint" exited with code 1

    $ bunx biome check src test
    Checked 78 files in 134ms. No fixes applied.

The gate fails while the code is clean, so **every** self-target worktree run
goes red at `gates` on lint alone, then burns its three repair rounds chasing
lint errors that do not exist. This is deterministic, not flaky.

## Root cause

`provisionWorktree` places the workspace at
`<factoryRoot>/runs/<runId>/workspace` — INSIDE the factory repo. A worktree's
`.git` is a file pointing at the main repo, so biome resolves the project root
upward to `<factoryRoot>` rather than treating the worktree as its own root.
The worktree's files then resolve to paths under `runs/`, which
`biome.json`'s `files.includes` excludes via `"!!**/runs"` — so every file is
ignored and biome exits non-zero on an empty file set.

The config is byte-identical in the worktree and on main; only the resolved
root differs. Non-self targets are unaffected: a cLens worktree also lives
under `<factoryRoot>/runs/`, but it loads cLens's own biome config, whose
ignore patterns do not mention `runs/`.

## Requirements

- [x] A red test that reproduces the vacuous pass/fail: running the factory's
      OWN lint gate from a path nested under a `runs/` ancestor must not
      report "no files processed". Pin the OBSERVABLE — an empty file set is
      never a legitimate lint result — rather than asserting a biome version's
      config-resolution behavior.
- [x] Fix so the self-target's lint gate reports on the worktree's real files.
      Weigh at pickup, do not assume:
      (a) anchor the ignore to the repo root (`/runs` rather than `**/runs`)
          so it cannot match an ancestor segment;
      (b) give the gate explicit paths (`biome check src test`), which is what
          demonstrably works today;
      (c) provision worktrees outside the factory repo.
      (a) is the smallest and keeps one gate command for every target; (c)
      changes the reclaim/breadcrumb story and is almost certainly too big.
- [x] Whatever is chosen, `bun run lint` must stay correct when run normally
      from the repo root — that path is not broken today.

## Verify

- [x] The red test fails before the fix and passes after.
- `bun run lint && bunx tsc --noEmit && bun test` from the repo root.
- A self-target worktree run reaches the `gates` node and lint reports on real
  files (a genuine lint error still fails it).

## Out of scope

The three load-sensitive test failures (`codexExecArgv`,
`ADW_DISABLE_TRACING`, the cLens hooks bag) — those are timeouts on tests
doing real git I/O and fail identically on main. Separate concern.

## Result (2026-09-11)

Fixed by candidate (a): `biome.json` `files.includes` `"!!**/runs"` →
`"!runs"` (written as `!runs/**`; biome's own formatter normalizes it). The
old pattern matched a `runs` segment anywhere in the ABSOLUTE path, so from a
worktree it matched the ancestor and ignored every file; the anchored form
matches only the repo-root `runs/`.

Measured both directions:

| cwd | before | after |
|---|---|---|
| repo root | 82 files | 83 files (82 + the new test) — `runs/` still excluded |
| run worktree | 0 files, exit 1 | 81 files, exit 0 |

Candidates (b) explicit gate paths and (c) provisioning worktrees outside the
repo were not needed — (a) is one line and keeps a single gate command for
every target.

`test/lint-gate.test.ts` uses the repo's own `node_modules/.bin/biome` by
absolute path: the fixture is a bare temp dir, so a `bunx biome` there
resolves nothing and the test would have failed on empty stdout rather than on
the bug. It also carries the no-regression guard (repo root must still never
descend into `runs/`), which passed before the fix and after.
