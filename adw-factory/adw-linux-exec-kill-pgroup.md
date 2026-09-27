---
id: adw-linux-exec-kill-pgroup
type: bug
status: done
priority: 1
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-linux-exec-kill-pgroup-1789133110470","branch":"adw/adw-linux-exec-kill-pgroup","workspace":"/Users/silouane/adw-factory/runs/adw-linux-exec-kill-pgroup-1789133110470/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# On Linux a timed-out exec never returns — the m4-06 ceiling does not hold

## Symptom

Found 2026-09-11 by CI's very first run (PR #4, job 103264873181,
ubuntu-latest). 804 pass, 1 fail:

    (fail) worktree exec — timeoutMs (adw-m4-06) > a command sleeping past
           timeoutMs is killed: non-zero code naming the timeout, and the
           process is actually gone [30955.89ms]

30955ms is the test budget, i.e. it HUNG. Two sibling tests in the same file,
using the same `freshWorkspace` fixture, passed in 30ms and 2039ms — so the
fixture is fine and the hang is specific to the timeout-kill path.

Green on macOS (838/0 locally). Linux-only.

## Root cause

`capture()` (`src/workspace/worktree.ts:244`) runs every exec as
`["sh", "-c", cmd]` and, on timeout, does:

    proc.kill("SIGKILL")          // line 263 — kills the PROCESS

then awaits

    Promise.all([ Response(proc.stdout).text(),
                  Response(proc.stderr).text(),
                  proc.exited ])

Whether that works depends on whether `sh` **execs** or **forks**:

- **macOS** — `sh` execs a single simple command, replacing itself. Verified:
  `sh -c "sleep 5"` reports the sleep PID == the shell PID. One process →
  SIGKILL kills it → both pipes reach EOF → the await resolves.
- **Linux (dash on ubuntu-latest)** — forks. SIGKILL kills only the shell;
  the orphaned child **inherits the stdout/stderr pipe write-ends**, so
  `Response(proc.stdout).text()` waits for an EOF that never arrives. In the
  failing test the orphan is `sleep 987654` — eleven days.

## Why this matters well beyond a red test

adw-m4-06 exists so that *"a wedged gate cannot hold the lane past the
wall-clock deadline"*. On Linux that guarantee is void: a wedged gate that
spawned any child hangs `exec` forever, and the node runs to the ENGINE
deadline (120 min) instead of its own ceiling. Every gate, every lane.

The factory has only ever run on macOS, so this was invisible until CI.

## To weigh at pickup — do NOT assume a design

- (a) **Kill the process group.** Spawn detached so the child leads its own
      group, then signal `-pgid`. Correct and portable; changes process
      semantics for every exec, so the blast radius is real.
- (b) **Stop awaiting pipe EOF after a timeout kill.** Return the timeout
      result and abandon the readers. Much smaller — but it LEAKS the orphan,
      and the existing test's `pgrep` assertion ("the process is actually
      gone") would still fail, correctly.
- (c) Both: (a) for correctness, plus a bounded wait so a pathological child
      can never wedge the reader.

(a) is the real fix; (b) alone does not satisfy the test that found this.

## Requirements

- [x] A red test that fails on Linux and passes after. The EXISTING test is
      already that test — it must go green on CI, not just locally.
- [x] `exec` with a `timeoutMs` returns within a bound on BOTH platforms, with
      no surviving child.
- [x] macOS behaviour unchanged (it is correct today).

## Verify

- CI green on ubuntu-latest — this is the acceptance check, and it cannot be
  confirmed locally on a Mac.
- `bun run lint && bunx tsc --noEmit && bun run test` locally.

## Out of scope

The container and E2B kinds have their own exec/timeout edges
(`container.ts`, `e2b.ts`). Check whether they share this shape, but fix them
in their own tickets.

## Result (2026-09-11) — fixed on main, CI-verified on Linux

Design (a): spawn `detached` so the child leads its own process group, and have
the timeout signal `-pid`. Falls back to a bare child kill on ESRCH so a raced
exit never throws. Switched to `node:child_process` (Bun.spawn cannot detach)
and awaits `close` rather than `exit`, so stdout/stderr are guaranteed drained.

**The bug was reproduced on macOS rather than needing a Linux runner.** `sh -c
"<cmd> & wait"` forces a fork on EVERY platform, so the failing contract — the
whole process TREE must die — is pinned by a test that runs anywhere. It hung
for the full budget before the fix and passes after. That test is now permanent,
so this cannot regress silently on either platform.

| | before | after |
|---|---|---|
| the original test, macOS | passed (sh execs) | 30s-hang → 1250ms |
| the new fork test, macOS | HUNG to budget | passes |
| full suite, CI ubuntu | 1 fail, 30.8s hang | **pass, 34s** |

Acceptance was CI green on ubuntu-latest, which could not be confirmed on a
Mac — verified on PR #5 and on main at `0a9a083`. Closed BY HAND: this was
fixed directly on main alongside two gh-adapter defects, not via its own run,
so no attempt exists for `sync-pr-state` to reconcile.

**Out-of-scope note still open:** the container and E2B kinds have their own
exec/timeout edges. They were NOT audited for this shape.

## The run that tried to build this (2026-09-11)

`adw-linux-exec-kill-pgroup-1789133110470` was launched after the fix had
already landed on `main` (`0a9a083`), so its worktree branched from a base
where the bug no longer existed. The bug lane wrote a reproducing test and
`red-check` ran it THREE times — the initial pass plus both revise rounds —
and every time all three gates PASSED:

    node-end red-check retry  gates: lint pass, typecheck pass, test pass
    round red-check 1 of 2
    node-end red-check retry  gates: lint pass, typecheck pass, test pass
    round red-check 2 of 2
    node-end red-check retry  gates: lint pass, typecheck pass, test pass

It then blocked, correctly: **a test that will not go red does not prove a
bug, so there is nothing to fix.** That is exactly what red-check exists to
enforce — no fix ships without a failure the machine watched happen. The lane
refused to manufacture work for an already-fixed defect.

The `blocked` status it wrote is therefore honest about that run. This ticket
is `done` because the defect is fixed and CI-verified, not because that run
succeeded.
