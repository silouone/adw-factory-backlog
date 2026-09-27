---
id: adw-tools-02-a-target-declares-the-commands-its-agents-may-run
type: feat
status: done
priority: 1
created: 2026-09-20
review: false
caps: {minutes: 120, turns: 500}
depends: []
attempts: [{"runId":"adw-tools-02-a-target-declares-the-commands-its-agents-may-run-1789897638104","branch":"adw/adw-tools-02-a-target-declares-the-commands-its-agents-may-run","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-tools-02-a-target-declares-the-commands-its-agents-may-run-1789897638104/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/98","provider":"claude","model":"sonnet"}]
---
# A target declares the commands its agents may run, so a Python target's agent can execute its own tests

> Measured 2026-09-20 on wave 1 of sabado (seven runs). Every agent ran
> under `permissionMode: "acceptEdits"` with `AGENT_ALLOWED_TOOLS =
> ["Bash(bun:*)", "Bash(bunx:*)"]` (`src/pipeline/nodes/build.ts:205`).
> On a Python + npm target that auto-approves nothing: `pytest`, `python3`,
> `just`, `alembic`, `npm`, `scripts/sabado`, even `/bin/echo hello` and
> `bash run.sh`, were refused with "This command requires approval" /
> "Contains shell syntax that cannot be statically analyzed", and there is no
> approver in a factory run. `sabado-12`'s test-only agent asked for 54 Bash
> calls and was allowed 26 (read-only ones), then stopped without writing
> its artifact — `blocked`. `sabado-13`'s agent, unable to run `alembic`,
> shipped an `env.py` change that makes `alembic upgrade head` apply nothing
> while printing every step (SQLAlchemy 2 autobegin swallows the migration
> transaction) — the gate caught it, three blind repair rounds did not.
> `sabado-17` likewise. The three green runs (`11`, `14`, `15`) were written
> blind and verified by the gates alone. `review: false`: hard-gated.

## What is being built

An optional target field, `allowedTools`: a list of SDK permission patterns
(`Bash(just:*)`, `Bash(pytest:*)`, …) **appended** to `AGENT_ALLOWED_TOOLS`
for every agent stage of a run against that target. Nothing is removed from
the factory's own list; `AGENT_TOOLS` (which tools exist) is untouched.

## Requirements

- [ ] **R1** `src/targets/loader.ts`: `allowedTools` joins `KNOWN_FIELDS`;
      an array of non-empty strings each matching `^[A-Za-z]+\(.+\)$`
      (the SDK's `Tool(pattern)` shape); anything else is a reported
      validation error naming the entry. Absent → not present on
      `TargetConfig`.
- [ ] **R2** `src/pipeline/nodes/build.ts`: the resolved `allowedTools`
      passed to the SDK is `[...AGENT_ALLOWED_TOOLS, ...target.allowedTools]`,
      deduplicated, order preserved. Both agent types (build, repair) and
      every lane's stages get it; the review node (no Bash) is unaffected.
- [ ] **R3** The journal's first agent `node-start` details carry the
      resolved list, so "what could this agent run" is answerable per run.
- [ ] **R4** The assembled prompt for a target with `allowedTools` gains one
      sentence: *run one plain command per Bash call — `cd`, `&&`, pipes,
      `$(…)` and `eval` are refused by the permission layer.* (The SDK's
      static analysis refuses compound commands regardless of the list.)
- [ ] **R5** `targets/sabado.json` declares:
      `Bash(just:*)`, `Bash(pytest:*)`, `Bash(python3:*)`, `Bash(python:*)`,
      `Bash(/Users/silouane/.sabado/venv-backend/bin/*)`, `Bash(alembic:*)`,
      `Bash(mypy:*)`, `Bash(lint-imports:*)`, `Bash(npm:*)`, `Bash(npx:*)`,
      `Bash(node:*)`, `Bash(scripts/*)`, `Bash(docker:*)`, `Bash(git:*)`.
      `.claude/skills/adwf/references/targets.md` documents the field.

## Verify

- [ ] Red test (`test/targets/loader.test.ts`): a valid list loads; an
      entry `pytest` (no parentheses) is an error naming it. RED today.
- [ ] Red test (`test/pipeline/nodes/build.test.ts`): with a target
      declaring `Bash(pytest:*)`, the SDK options' `allowedTools` equals
      `["Bash(bun:*)", "Bash(bunx:*)", "Bash(pytest:*)"]`; with no field it
      is unchanged. RED today.
- [ ] Red test: the assembled prompt contains the one-command-per-call
      sentence only when the target declares the field.
- [ ] Existing build/lane suites green unmodified.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] **Operator, after merge:** one sabado bug ticket re-fired; its capture
      shows `pytest` calls with `PostToolUse` events (they ran).
