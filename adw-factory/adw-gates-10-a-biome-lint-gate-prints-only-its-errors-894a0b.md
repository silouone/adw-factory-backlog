---
id: adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b
type: chore
status: blocked
priority: 2
created: 2026-10-02
caps: {minutes: 60, turns: 150}
depends: []
attempts: [{"runId":"adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b-1790968675723","branch":"adw/adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b-1790968675723/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b-1790979182664","branch":"adw/adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-10-a-biome-lint-gate-prints-only-its-errors-894a0b-1790979182664/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# A Biome lint gate prints only its errors, so repair can see the one that fails it

## Why

Biome prints at most **20 diagnostics** by default, and warnings count toward that limit. silou-hq carries 57 pre-existing `noNonNullAssertion` warnings, and adw-factory has its own.

In hq-07's round-3 repair prompt (`runs/hq-07-search-inspector-and-full-screen-in-preact-a2c261-1790925359587/prompts/0006-repair.txt`):
- the gate excerpt reads `Found 1 error. Found 57 warnings. … Diagnostics not shown: 39.`;
- the stderr tail is 20 warnings;
- the one error (`test/web/fetch-text.test.tsx format`) appears **0 times**, because it was among the 39 diagnostics not printed.

The agent was told lint failed but never told where. No change to `REPORT_TAIL_CHARS` (6000, `src/pipeline/nodes/gates.ts:102`) can fix this: the diagnostic was never printed.

## What to build

Change the lint gate `cmd` in every target whose lint is Biome:
- `targets/adw-factory.json`
- `targets/silou-hq.json`
- `targets/clens.json`, after confirming its `bun run lint` is Biome

The new command:

```
bunx biome check --diagnostic-level=error --max-diagnostics=none .
```

Pass/fail semantics are unchanged. Biome warnings never fail `check` unless `--error-on-warnings` is set, and `--diagnostic-level` only filters what is printed. Measured on Biome 2.5.15 in the hq-07 workspace (2026-10-02):

| tree | default `biome check .` | errors-only cmd | output |
|---|---|---|---|
| clean | exit 0 | exit 0 | 45 bytes |
| one misformatted file | exit 1 | exit 1 | 821 bytes, names the file |

Before switching each target, check that its `package.json` `lint` script has no extra arguments (paths, `--error-on-warnings`) that the new command would drop. Carry them over if it does.

The documented `just verify` and `bun run lint` stay as they are. This only changes what the gate prints for repair. Note it beside the adw-gates-04 decision ("the documented verify command is not the gate").

## Red first (Art. I)

Add a loader test for each changed target: its lint gate `cmd` contains `--diagnostic-level=error` and `--max-diagnostics=none`.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`
- On each changed target's base commit, run the new cmd: exit 0 and short output.
- Then add one misformatted file: exit 1, and the file is named in the last 6000 chars of the output.
