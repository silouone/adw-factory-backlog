---
id: sabado-14-nothing-red-or-drifted-lives-on-main
type: feat
status: done
priority: 2
created: 2026-09-19
caps: {minutes: 300, turns: 1000, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-14-nothing-red-or-drifted-lives-on-main-1789850372229","branch":"adw/sabado-14-nothing-red-or-drifted-lives-on-main","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-14-nothing-red-or-drifted-lives-on-main-1789850372229/workspace","outcome":"blocked"},{"runId":"sabado-14-nothing-red-or-drifted-lives-on-main-1789859138376","branch":"adw/sabado-14-nothing-red-or-drifted-lives-on-main","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-14-nothing-red-or-drifted-lives-on-main-1789859138376/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1221","provider":"claude","model":"sonnet"}]
---
# ci: main is checked after every merge, a stale migration branch goes red before it, and nobody pushes main by hand

> **Audit:** **P1-16** (axes I, F, M) + the `schema.d.ts` drift (A P2) —
> the second half of Lane B, running beside `sabado-13`. Review stays on:
> the shape of a CI job is a judgement.

## What happens today

- `ci.yml:11-13` runs on `pull_request` only; `deploy-staging.yml:68-135`
  gates on the workflow runs of the **PR head** — which pre-date the merge.
  Nothing ever checks the merged tree.
- Branch protection is a 403 on the free plan; `ci.yml:9` still says "that
  is what the branch protection requires".
- Incidents: `de15391a` (#659, 2026-08-29) landed a revision whose
  `down_revision` named a head `main` had moved past → two heads on `main`,
  **4.5 h of frozen staging and red CI on every open PR**. Again 2026-09-16:
  `f81d522c` (#1118) and `625cada4` (#889) both added a `0064_*` the same
  day. Direct pushes `615193fe` (16 files) and `8150fcf7` (12 files, a
  feature) reached `main` with no PR and were deployed by the next merge.
- `frontend/src/api/schema.d.ts` is regenerated from the OpenAPI document
  by habit (35 times, 149 of 179 API commits skipped it); no check.
- `ci.yml:159-161` and `BACKEND.md:19-21` still teach that a two-head
  `alembic upgrade head` fails "silently" — fail-loud since #595
  (`backend/Dockerfile:38`, `&&`).

## Requirements

- [ ] **R1 — a check on `main` after the merge.** A job triggered on
      `push: branches: [main]` (in `ci.yml` or a sibling workflow) running,
      on the merged tree: `alembic heads` = 1; no **new** duplicate numeric
      prefix (the four known — `0045 0046 0050 0064` — are the baseline the
      script carries); `lint-imports`; regenerate `schema.d.ts` from the
      OpenAPI document (the repo's own `generate:api` path or its offline
      equivalent) and `git diff --exit-code frontend/src/api/schema.d.ts`.
      Red → the existing Slack webhook is told, the way deploy failures are.
- [ ] **R2 — a stale branch goes red before it merges.** In the PR `back`
      job: for every migration file **added** by the PR
      (`git diff --name-only $BASE_SHA...$HEAD_SHA -- backend/alembic/versions`),
      its `down_revision` must be the head of the **base** commit's chain;
      otherwise fail, naming both revisions.
- [ ] **R3 — the staging gate also refuses a direct push under a merge.**
      In `deploy-staging.yml`, the merge commit's first parent must itself
      be a PR merge commit (`/commits/<parent>/pulls` non-empty) or the
      previously deployed SHA.
- [ ] **R4 — the local guard.** `scripts/bootstrap` installs a `pre-push`
      hook that refuses `git push origin main` (opt-out with
      `SABADO_ALLOW_PUSH_MAIN=1`, and the hook says so). `sabado-00` landed
      first; this ticket adds only the hook lines.
- [ ] **R5 — stale sentences.** Delete `ci.yml:9`'s branch-protection
      claim, rewrite `ci.yml:159-161` and `BACKEND.md:19-21` to the
      fail-loud truth.
- [ ] **R6 — scripts, not YAML prose.** R1's checks and R2's check are two
      scripts under `scripts/ci/` that run locally with documented
      arguments; the workflows only call them. That is what makes the
      Verify block below executable without GitHub.

## Files

`.github/workflows/ci.yml` (three regions: the header comment; one step in
the `back` job right after `alembic heads`; a new job at the end — nothing
else; `sabado-17` and `sabado-23` do not touch this file) ·
`.github/workflows/deploy-staging.yml` · `scripts/ci/main-check.sh` (new) ·
`scripts/ci/alembic-branch-check.sh` (new) · `scripts/bootstrap` (hook
lines only) · `.claude/skills/sabado-project/BACKEND.md` (lines 19-21 only;
`sabado-11` appends a rule elsewhere in the file).

## Verify

- [ ] `scripts/ci/main-check.sh` on HEAD → exit 0, printing each check.
- [ ] `scripts/ci/alembic-branch-check.sh <base-sha> <head-sha>` on a
      scratch branch carrying a migration whose `down_revision` is
      `0066_…` → non-zero, naming `0066_…` and the real head. Same script
      on this ticket's own branch → exit 0.
- [ ] A scratch branch that edits one route's response schema without
      regenerating `schema.d.ts` → `main-check.sh` non-zero on the diff step.
- [ ] `git push origin main` from a bootstrapped clone → refused by the hook
      with the opt-out named; `SABADO_ALLOW_PUSH_MAIN=1` → the hook steps
      aside (do **not** actually push).
- [ ] `grep -n "branch protection" .github/workflows/ci.yml` → nothing;
      `grep -n silent .claude/skills/sabado-project/BACKEND.md` → nothing
      about migrations.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

The `alembic check` test (`sabado-13`); `just merge`, PR template, CODEOWNERS
(§5 #10); turning merge commits off (a repo setting — operator); any change
to the GitHub plan.
