---
id: adw-pr-01-resolve-a-card-to-its-pull-request
type: feat
status: done
priority: 2
created: 2026-09-17
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-pr-01-resolve-a-card-to-its-pull-request-1789678281034","branch":"adw/adw-pr-01-resolve-a-card-to-its-pull-request","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-pr-01-resolve-a-card-to-its-pull-request-1789678281034/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-pr-01-resolve-a-card-to-its-pull-request-1789714981882","branch":"adw/adw-pr-01-resolve-a-card-to-its-pull-request","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-pr-01-resolve-a-card-to-its-pull-request-1789714981882/workspace","outcome":"blocked","provider":"claude","model":"sonnet","pr":"https://github.com/silouone/adw-factory/pull/75"}]
---
# Resolve a card to its pull request — the pure core, before any network

Spec: `specs/adw-v1.9-pr-state.md` (Stage 1). Investigation and every measured
figure: `ai_docs/2026-09-17-github-ci-review-on-card.md`.

Stage 1 of three. **No I/O, no UI, nothing visible on the board when this
lands.** It builds the four pure derivations the next two stages stand on, and
it is fully verifiable with no web work in existence — the same staging
precedent `adw-v1.2` set for its heartbeat and persistence stages.

## Why these four, and why first

The board cannot name the PR belonging to a card. `open-pr.ts:709` returns
`{kind:"next", patch:{pr: prUrl}}` — the URL goes into `ctx.data` and dies with
the process. **No journal event carries it.** But branches are deterministic
and `gh pr list` returns `headRefName`, so the join needs no new run data and
works retroactively on every banked run.

Each piece below is a pure function. That is the point: they are the whole
decision surface of the feature, and none of them needs a network to be proven
right.

## 1. `github: "owner/name"` on the target config

`TargetConfig` (`src/targets/loader.ts:50`) records `repo` as a **local
path**. `gh` resolves a repo from the cwd's git remote, which requires the repo
to be cloned — and **none of the 7 Coorpacademy targets exists on this
machine**. Without an explicit identity the whole feature is unreachable for
exactly the repos that motivated it.

- [ ] Optional `github?: string` on `TargetConfig`, joining `sandbox`,
      `provider` and `systemPrompt` as an optional run-scoped field.
- [ ] Validated as `owner/name`: exactly one `/`, neither side empty, no
      whitespace, no scheme, no trailing `.git`. A malformed value is a
      **loud target-config rejection** carrying the field and the offending
      value (Art. IX), never a silent fallback to derivation.
- [ ] Absent is valid and is not an error — the 4 cloned targets must keep
      loading byte-identically, with no edit to their JSON.

## 2. The repo-identity resolver

- [ ] `github` present → use it verbatim.
- [ ] Absent → derive from the target repo's `origin` remote URL. **The git
      call is injected** (`(repoPath) => string | undefined`), so this module
      stays pure and the suite never shells out.
- [ ] Parse both real forms: `git@github.com:owner/name.git` and
      `https://github.com/owner/name(.git)`. Strip `.git`.
- [ ] Neither available, or the URL is not GitHub → `undefined`. **Never a
      guessed identity.** `undefined` is what Stage 2 turns into "no chips for
      this target plus a notice"; a fabricated `owner/name` would produce
      confident wrong data, which is worse than none.

## 3. One check classifier, two input shapes

Today `ci-round.ts::parseChecks` reads `gh pr checks`' `state` field. The
board will read `statusCheckRollup`, which has **no `state` field at all** —
it carries `status` + `conclusion`. Feeding the rollup to `parseChecks`
classifies **every check as passing**, silently.

- [ ] One pure classifier both shapes normalize into, yielding the existing
      `ChecksView` (`passing | failing | pending`, plus `failedRunId`).
- [ ] Rollup mapping, **settled by operator grilling 2026-09-17 — do not
      re-litigate in code**:
      - `conclusion` ∈ `FAILURE` / `TIMED_OUT` / `STARTUP_FAILURE` /
        `ACTION_REQUIRED` → **failing**
      - `conclusion` = `CANCELLED` → **cancelled**, its own state. A cancelled
        run tested nothing, so it is never green — but it is not a test
        failure either, and calling it one would send you hunting a bug that
        does not exist.
      - `conclusion` ∈ `SKIPPED` / `NEUTRAL` → **non-failing**. Path filters
        and matrix skips are normal and must not downgrade a green.
      - else any `status` ∈ `QUEUED` / `IN_PROGRESS` / `PENDING` → **pending**
      - else → **passing**
- [ ] An **empty** rollup is its own verdict — **`no-checks`** — and must not
      read as passing. "No CI configured" and "CI passed" are different facts.
      Note `interpretGh` already treats gh's "no checks reported" stderr as
      data, not an error.
- [ ] `ChecksView.status` therefore widens to five:
      `passing | failing | cancelled | pending | no-checks`. Every existing
      `ci-round` call site must be re-checked against the widened union —
      `tsc` will find them, but the *semantics* are yours: decide explicitly
      whether `cancelled` and `no-checks` should trigger a CI repair round the
      way `failing` does. **They should not** — a repair agent cannot fix a
      cancelled run or absent CI.
- [ ] `parseChecks` is refactored onto the shared rule, not left as a second
      copy (Art. VIII). **The board and the CI-repair round must never
      disagree about whether a PR is red.**
- [ ] `ci-round`'s existing suite stays green with **no assertion changes**.

## 4. The branch → ticket join

- [ ] Given a branch name, a `branchPrefix`, and the set of ticket ids the
      board knows, return the matching ticket id or `undefined`.
- [ ] Handles the retry suffix: `nextAttemptBranch`
      (`src/workspace/worktree.ts:68`) yields `<prefix><id>`, then
      `<prefix><id>-2`, `-3`, …
- [ ] **Match by longest prefix against the known id set — never by regex
      suffix strip.** A ticket id that legitimately ends in `-2` would
      otherwise bind to a different ticket's branch, silently and
      unfalsifiably. This is the single most important test in this ticket.
- [ ] Two PRs on the same ticket → the highest PR number wins.
- [ ] An unknown branch, or a branch without the prefix → `undefined`.

## TDD (Art. I — non-negotiable)

Every item above is a pure function: arguments in, value out, no clock, no
network, no filesystem. Write the tests first, **confirm them red, present
them for review, then implement to green.** No source before a reviewed red
test. The `-2`-suffix trap and the rollup↔`gh pr checks` agreement test are
the two that must exist before any implementation.

Real fixtures beat invented ones. `gh pr list --state all --limit 100 --json
number,state,headRefName,statusCheckRollup,reviewDecision` against this repo
returns 60 real PRs; bank a trimmed slice of that payload as the fixture
rather than hand-writing shapes.

## Verify

- [ ] Every new pure suite green.
- [ ] `test/targets/loader.test.ts` green, including a target with no
      `github` field loading unchanged.
- [ ] `test/pipeline/nodes/ci-round.test.ts` green with no assertion changes.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Any `gh` call, any cache, any UI, `runWeb` target awareness — all Stage 2
(`adw-pr-02`) and Stage 3 (`adw-pr-03`). Do not add the field to any
`targets/*.json` yet; the resolver's fallback must be exercised first.
