---
id: hq-04-the-landing-answers-in-under-a-second-6ded62
type: feat
status: in-progress
priority: 2
created: 2026-10-01
depends: [hq-02-the-read-only-http-surface-is-tested-72e739]
attempts: [{"runId":"hq-04-the-landing-answers-in-under-a-second-6ded62-1790898726950","branch":"adw/hq-04-the-landing-answers-in-under-a-second-6ded62","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-04-the-landing-answers-in-under-a-second-6ded62-1790898726950/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/6","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The landing answers in under a second, whatever the slow sources do

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Spec: the landing renders from the last built graph; slow sources (the M1 over ssh, the factory,
`claude agents`) refresh in the background and never block a request.

## Red first

- With an injected source that never resolves, `/graph.json` still answers from the last good graph, well within budget.
- A failing rebuild keeps serving the previous graph and records the failure (it doesn't crash or blank the page).

## Acceptance criteria

- [ ] Build timings per source are logged at startup and on each rebuild.
- [ ] On the real machine, a cold page load of `/` plus `/graph.json` takes under 1s once the first build exists (measured and noted in the PR).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-02-the-read-only-http-surface-is-tested-72e739
