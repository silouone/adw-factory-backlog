---
id: adw-gates-config-error-regex
type: bug
status: done
priority: 1
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-gates-config-error-regex-1789113047141","branch":"adw/adw-gates-config-error-regex","workspace":"/Users/silouane/adw-factory/runs/adw-gates-config-error-regex-1789113047141/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-gates-config-error-regex-1789122645774","branch":"adw/adw-gates-config-error-regex","workspace":"/Users/silouane/adw-factory/runs/adw-gates-config-error-regex-1789122645774/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# A test NAME containing "command not found" blocks the run with zero repair rounds

## Symptom

`adw-m5-06-remote-capture-parity-1789085359632` blocked at `gates` after 89
minutes of good work:

    gate "test" command "bun test" could not execute (exit 1)
      — target configuration error, not a gate failure

`bun test` did execute. It exited 1 because the suite has pre-existing
failures. No repair round was charged, and the `node-end` event carried no
gate details at all — `just gates <runId>` returns nothing — so the reason was
only recoverable by reading `gates.ts`.

## Root cause

`isConfigError` (`src/pipeline/nodes/gates.ts:137`) classifies a gate as
un-runnable when:

    exec.code === 127 || exec.code === 126 ||
    /command not found|No such file/.test(exec.stderr)

The exit code here was **1**, so the stderr regex fired. `bun test` writes test
NAMES to stderr, and two of this repo's names contain the literal string:

    gates node — target-configuration errors (E3) > exit code 127
      (command not found) → fail naming gate, cmd, 'target configuration'
    gates node — target-configuration errors (E3) > stderr matching
      /command not found/ (independent of the code) → target-configuration fail

They are the tests for this very heuristic. **The classifier is tripped by the
names of the tests that test the classifier.** Verified: `bun test
test/pipeline/nodes/gates.test.ts 2>/tmp/e >/tmp/o` puts 2 occurrences on
stderr, 0 on stdout.

Deterministic, not flaky: every non-zero `bun test` exit on this repo blocks
terminally instead of routing to repair.

## Why it matters beyond the string

Matching stderr CONTENT cannot distinguish "the shell could not find the
binary" from "a process printed that phrase". The consequence is severe and
asymmetric — a terminal block with no repair round — so the test for it should
be correspondingly strong. E3's intent (don't charge repair rounds for a
misconfigured target) is right; the implementation over-reaches.

## Requirements

- [ ] Red test: a gate whose command RUNS and exits non-zero, but whose stderr
      contains `command not found` (e.g. it echoes a test name), must route to
      `retry` — not a `fail`. This is the exact production case.
- [ ] Keep the true positives green: exit 127 and exit 126 still classify as
      config errors (existing tests must stay passing).
- [ ] Narrow the heuristic. Weigh at pickup, do not assume:
      (a) trust the exit codes only (127/126) and drop the stderr regex —
          simplest, loses coverage for wrappers that remap the code;
      (b) require the stderr pattern AND a suspicious exit code;
      (c) anchor the pattern to a shell-emitted shape (line-start
          `<cmd>: command not found`) rather than a substring anywhere.
- [ ] Journal the reason on the terminal-fail path. Today `fail()` returns
      before any gate result is recorded, so a blocked run states no cause in
      its own journal (Art. VI: a run that cannot be replayed is defective).

## Verify

- The red test fails before and passes after; the 127/126 tests stay green.
- `bun run lint && bunx tsc --noEmit && bun test`.
- A self-target run whose suite is red reaches `repair`, not a terminal block.

## Out of scope

Making the suite itself green — separate concern, and the reason this defect
fires at all. The two compound: each is survivable alone.

## Run log

**2026-09-11 — run `…-1789113047141` blocked at `base-green-check`** (6 min,
before any agent work): `gate "test" failed on the untouched checkout (exit 1)
— target base not in a provable-clean state`. The bug lane refused to build a
fix on a base it could not prove green. That is the lane working correctly.

The base was red for the reason fixed in `bfa753f` (a 5000ms per-test default
against integration tests doing real git/docker on a loaded workstation), not
for anything to do with this ticket. Reset to `queued`: the base is now green
(838 pass / 0 fail), so this is buildable.

**2026-09-11 — MERGED.** PR #2 squash-merged as `7ea75e9` (+59/-10).

Closed BY HAND. `sync-pr-state` cannot reconcile this one: it only acts on
`in-review` tickets with a PR recorded in `attempts:`, and this ticket's only
attempt is the BLOCKED factory run. That entry is left as-is — it records what
happened. The PR was opened by hand because of the bootstrap deadlock: the
reproducing test must contain `command not found` to prove the defect, which
put it on stderr, which made the unfixed classifier reject the change. The
factory could not land this fix by construction.

Verified after merge: a green suite's stderr (3 occurrences of the phrase) is
no longer classified as a config error, while `bash: badgate: command not
found` and `sh: tsc: No such file or directory` still are.
