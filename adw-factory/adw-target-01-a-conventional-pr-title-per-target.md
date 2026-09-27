---
id: adw-target-01-a-conventional-pr-title-per-target
type: feat
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 120, turns: 500}
depends: []
attempts: [{"runId":"adw-target-01-a-conventional-pr-title-per-target-1789810397605","branch":"adw/adw-target-01-a-conventional-pr-title-per-target","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-target-01-a-conventional-pr-title-per-target-1789810397605/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/89","provider":"claude","model":"sonnet"}]
---
# A target can ask for conventional-commit PR titles, so sabado's title lint stops rejecting every factory PR

> **Why now.** `targets/sabado.json` is live and its backlog is seeded
> (`~/personal_project/SABADO/sabado/tickets/`). sabado's
> `.github/workflows/pr-title.yml` runs `amannn/action-semantic-pull-request`
> on every PR: the title must be `type(scope)?: subject` with `type` in
> `feat perf fix revert refactor chore ci build docs test style`. The
> factory's `open-pr.ts::prTitle` renders `<ticketId>: <headline>` (or
> `adw: <ticketId>`), so **every** sabado PR would carry a red `PR title`
> check; sabado's `deploy-staging.yml` then refuses to deploy the merge, and
> the factory's own CI round (`src/github/checks.ts`: one failing entry
> fails the rollup) would burn its single repair round on a title no code
> change can fix. `review: false`: every Verify bullet is a red test or a
> gate.

## What is being built

An optional target field, `prTitle`, read by `open-pr` from the target it
already carries on `ctx.data.target` (the same structural subset idiom
`gates.ts` and `open-pr.ts:161` use).

- `"default"` (or absent): today's title, byte-identical. The existing test
  "the PR title carries more than the id" stays green unmodified.
- `"conventional"`: if the ticket's first headline already **is** a
  conventional subject (`^[a-z]+(\([^()]*\))?!?: \S`), the title is that
  headline, verbatim — the ticket author owns type and scope, and the
  target's CHANGELOG line stays clean. Otherwise the title is
  `<type>(<ticketId>): <headline>` with the lane mapped `chore → chore`,
  `feat → feat`, `bug → fix`; with no headline at all, `<type>(<ticketId>):
  <ticketId>`.

Traceability from PR to ticket does not depend on the title: the branch
name and the PR body (`prompts/pr-body.md`) carry the ticket id.

## Requirements

- [ ] **R1** `src/targets/loader.ts`: `prTitle` joins `KNOWN_FIELDS`;
      accepted values `"default" | "conventional"`; anything else is a
      reported validation error naming the field and the value (the
      `provider` precedent). Absent → not present on `TargetConfig`.
- [ ] **R2** `src/pipeline/nodes/open-pr.ts`: `prTitle` becomes a pure
      function of `(ticketId, ticketType, headline, style)`; the node
      reads `style` off `ctx.data.target.prTitle`.
- [ ] **R3** The conventional-subject regex is exported and tested on the
      four shapes: `fix(auth): …`, `ci: …`, `feat!: …`, and a plain
      sentence (not conventional).
- [ ] **R4** `targets/sabado.json` gains `"prTitle": "conventional"`.
      `.claude/skills/adwf/references/targets.md` documents the field in
      one row.
- [ ] **R5** No other target changes; `targets/clens.json` and
      `targets/adw-factory.json` keep the default.

## Verify

- [ ] Red test (`test/targets/loader.test.ts`): `prTitle: "conventional"`
      loads; `prTitle: "semantic"` is an error naming `prTitle`. RED today
      (unknown field).
- [ ] Red test (`test/pipeline/nodes/open-pr.test.ts`): with
      `prTitle: "conventional"` and a ticket whose body opens with
      `# fix(calendar): recurring events keep their Paris wall-clock`, the
      `gh pr create --title` argument is exactly that line. RED today.
- [ ] Red test: same style, a `bug` ticket whose headline is `The baseline
      outlives its environment` → `fix(<ticketId>): The baseline outlives
      its environment`. RED today.
- [ ] Red test: a `chore` ticket, no headline → `chore(<ticketId>):
      <ticketId>`. RED today.
- [ ] The existing default-style test passes unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

A per-ticket `scope:` field; PR body changes; anything in
`sync-pr-state`/`branch-ticket`.
