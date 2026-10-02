---
id: adw-gates-09-the-engine-runs-the-formatter-not-an-agent-6b2d63
type: feat
status: in-review
priority: 2
created: 2026-10-02
caps: {minutes: 150, turns: 400}
depends: []
attempts: [{"runId":"adw-gates-09-the-engine-runs-the-formatter-not-an-agent-6b2d63-1790928400339","branch":"adw/adw-gates-09-the-engine-runs-the-formatter-not-an-agent-6b2d63","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-09-the-engine-runs-the-formatter-not-an-agent-6b2d63-1790928400339/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/177","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The engine applies the target's formatter itself; formatting is never an agent's job

## Why

Art. III: "never an agent where a function suffices." Formatting is a function: `biome check --write` is deterministic and idempotent. Yet today the only way a format diff gets fixed is an agent running the formatter. When the agent can't (no shell, see adw-bug-40) or doesn't, a whole repair round goes to it:

- **hq-07, both runs:** each of the six repair rounds faced a format-only lint failure. The second run's agents hand-formatted the output, getting from 17 errors to 11 to 1, and never closed it. A salvage `biome format --write` fixed it in 51 ms.
- **hq-05:** repair rounds swung between lint and tsc, because a tsc fix re-broke Biome formatting (memory note, 2026-10-01).
- **adw-merge-01 (PR #162):** the salvage started with "format".

## What to build

1. **`TargetConfig.fix?: { cmd: string }`** (`src/targets/loader.ts`).
   - It is a single command that applies safe automatic fixes in place. For Biome: `bunx biome check --write .`. **Never** `--unsafe`.
   - Validated like a gate `cmd`: a non-empty string, otherwise a loud load error naming `fix.cmd`.
   - **Absent → byte-identical to today.**
2. **A deterministic `fix` step in the engine.**
   - It runs `workspace.exec(fix.cmd)` immediately before every `gates` pass that follows an agent node: after build/test, and after each repair round.
   - It does **not** run before the baseline. The base is measured as-is.
   - The fix step's exit code is never a gate. Lint still decides.
   - A non-zero exit is journaled (`fix-step` event with exit code and tail) and the run continues to gates. A fix that can't apply everything leaves the rest for lint to report.
   - Its file changes go into the run's diff like any agent edit, and the journal records which files it touched (`git diff --name-only` before vs after).
3. **Targets:** add `fix` to `targets/adw-factory.json`, `targets/silou-hq.json` and `targets/clens.json`. Check that clens's lint is Biome before adding it.
4. **Tell agents in the prompt:** "the engine formats your changes before gates. Do not spend turns on formatting."

## Spec amendment

Add the `fix` step to the pipeline graph in the gates section of `specs/adw-v1.md` and the lanes spec. Cover:
- where it runs and where it never runs (the baseline);
- that it never gates;
- why it is not an agent (Art. III).

## Red first (Art. I)

- loader: `fix` absent → unchanged; valid → kept; `fix: {}`, an empty `cmd` or a non-string → load error naming `fix.cmd`.
- engine, with a fake workspace recording `exec` calls:
  - with `fix` declared, the call order is `[fix, lint, typecheck, test]` after build and after each repair;
  - the baseline's calls contain no `fix`;
  - `fix` exiting 1 → the `fix-step` event is journaled and the gates still run;
  - without `fix`, the exec sequence is byte-identical to today.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`
- Manual check: copy silou-hq at 8164e67, add a misformatted line, run the gates node with `fix` declared. Lint must pass with no agent involved.
