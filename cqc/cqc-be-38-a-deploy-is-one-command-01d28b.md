---
id: cqc-be-38-a-deploy-is-one-command-01d28b
type: chore
status: queued
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# A deploy is one scripted command that a CI box can run

Sources: BE-18 (`docs/backend/decisions-2026-09-26.md:83`), `docs/backend/todo.md:53`;
`scripts/runner/README.md:19-27` (today's manual recipe). This is the in-repo half. The
`coorpacademy-infra` entry is `cqc-be-40`.

## Today

- No CI.
- `backend/package.json:8-18` has no `deploy` script.
- No `.nvmrc`; `engines` is `>=22.0.0`.
- The runner image and the stack deploy are two hand-run steps, and `runnerImageTag` defaults
  to `bootstrap` (`serverless.yml:35`). The dev stack was deployed that way on 2026-09-30.

## Change

- Add `deploy:dev`, `deploy:staging` and `deploy:prod` in `backend/package.json`. Each:
  1. builds and pushes the runner image from the repo root, tagged with the current 40-hex
     commit;
  2. runs `serverless deploy --stage <s> --param runnerImageTag=<commit>`.
- Refuse a dirty tree. The tag must name code that exists.
- Add an `.nvmrc` (Node 22), for the `n auto` step in the shared CodeBuild template.
- The shared template runs `cd backend && ${InstallCommand} && ${TestCommand}`.
  `contracts.ts:1,5` import `../../../poc0` and `../../../scripts/context-extract`, so the
  checkout must be the full repo. Document that.

## Acceptance criteria

- [ ] `npm --prefix backend run deploy:dev` does the whole thing on a clean tree.
- [ ] A test pins the three script names and the dirty-tree refusal (a script-level test is
      enough).
- [ ] `scripts/runner/README.md` points at the scripts instead of the hand recipe.

## Blocked by

- (nothing)
