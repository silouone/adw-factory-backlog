---
id: adw-gates-08-live-infra-tests-gate-only-what-touches-them-3ab83e
type: feat
status: done
priority: 1
created: 2026-10-02
caps: {minutes: 150, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-gates-08-live-infra-tests-gate-only-what-touches-them-3ab83e-1790899916500","branch":"adw/adw-gates-08-live-infra-tests-gate-only-what-touches-them-3ab83e","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-08-live-infra-tests-gate-only-what-touches-them-3ab83e-1790899916500/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/175","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The live Docker and E2B suites gate only the runs whose diff touches them

Operator decision **D-test-1, 2026-10-02: option C (scoped)**. Source: `ai_docs/2026-10-02-test-suite-time-audit.md` §3 #5/#10.

## Why

The adw-factory target's `test` gate is `bun run test`, the whole suite. That includes four files driving **live infrastructure**:

| file | measured (full suite, loaded) | drives |
|---|---|---|
| `test/cli.container.test.ts` | 105 s / 8 | real Docker + the `adw-agent` image |
| `test/workspace/container.contract.test.ts` | 64 s / 36 | real Docker |
| `test/workspace/container.reattach.test.ts` | 10 s / 10 | real Docker |
| `test/workspace/e2b.contract.test.ts` | 16 s / 4 | real E2B sandboxes. Billed. Runs whenever `E2B_API_KEY` is set. |

A gate pass happens at the baseline, after the build, after each repair round and after a rebase. Every one of them pays about 200 s for these files, plus their flakiness.

On 2026-10-02, under load, `cli.container` failed **4 of 8** with only `expect(code).toBe(0)`, received `1`. A failure like that can block a run, or get a base cached as red, for a change that never touched a container.

`test/workspace/e2b.test.ts` is **not** live. Its `E2bOps` is faked. It stays in the `test` gate.

## What to build

1. **`Gate.paths?: readonly string[]`** (`src/targets/loader.ts`): glob patterns, relative to the repo root.
   - Validated like `scopedCmd`. If present it must be a non-empty array of non-empty strings. A malformed value is a loud load error naming `gates[i].paths`.
   - **Absent → byte-identical to today.** cmc, sabado, clens, claude-home and silou-hq are unaffected.
2. **The `gates` node honours it.**
   - A gate with `paths` runs only if the run's changes touch at least one matching path. "Changes" means committed changes relative to the base, plus any uncommitted changes in the workspace. Gates run before the commit node, so the uncommitted part matters.
   - Otherwise the gate is recorded as **skipped**, never dropped silently (Art. V). The journal and the gate summary carry `skipped: true` and a reason, e.g. `paths [...] untouched by this run`.
   - A skipped gate counts as not-failing for the green decision.
   - Repair and ci-round re-evaluate the decision on each pass. A repair that starts touching `src/workspace/container.ts` brings the live gate in.
3. **The `baseline` node skips `paths` gates.**
   - There is no diff at the baseline, so a path-scoped gate is not snapshotted. Record it as skipped in the baseline record, so the cache shape stays explicit.
   - A run that does trigger the gate has no baseline for it. Any failure there is attributed to the run. That is the honest default for a gate that only runs when you touched its code. State this in the spec amendment.
4. **The self-target** (`targets/adw-factory.json`):
   - `test`: the current cmd plus `--path-ignore-patterns` for the four live files above. Check how bun 1.4.2 takes multiple patterns.
   - `test-live` (new), `paths`:
     - `src/workspace/container*`, `src/workspace/e2b*`, `src/workspace/types.ts` (the shared Workspace interface)
     - `containers/**`
     - `test/cli.container.test.ts`, `test/workspace/container*`, `test/workspace/e2b.contract.test.ts`, `test/workspace/docker-gate.ts`, `test/workspace/e2b-gate.ts`
   - The live files' cmd uses its own junit report path (`.adw/report-live.xml`), so it doesn't overwrite the `test` gate's report.
   - The files exist as of 2026-10-02: `container.ts`, `e2b.ts`, `types.ts`; `containers/Dockerfile`, `containers/e2b-template.ts`.
5. **`red-check` is unchanged.** Its `scopedCmd` narrowing applies to the `test`-named gate only.

## Spec amendment (Amendment rule, land it in the same PR)

- **`specs/adw-v1.13-run-economics.md` story 14** says the `gates` node runs "the real suite". Amend it: for a target that declares `paths` on a gate, the definition of done is "every gate that applies to the change". `just verify` (lint, tsc, the full `bun run test`) remains the operator's whole-suite check before merging.
- Add a short `Gate.paths` section beside the existing `scopedCmd` decision in `specs/adw-v1.1-lanes.md` (~:530). Cover the semantics, the skipped journaling and the baseline rule from item 3.
- **The accepted risk, stated in writing.** A change outside `paths` (for example in `src/cli.ts` or a lane) can break the container lane without the gate seeing it. Two mitigations:
  - the faked container/e2b unit tests still run in `test`;
  - `just verify` before merge.

## Red first (Art. I)

- loader: `paths` absent → gate unchanged; a valid `paths` is kept; `paths: []`, a non-array or a non-string entry → load error naming `gates[i].paths`.
- gates node, with a fake workspace:
  - changed files miss every pattern → the gate's `exec` is never called, and the journal shows it skipped with its reason;
  - one changed file matches → it runs;
  - an uncommitted change counts.
- baseline: a `paths` gate is never executed at the baseline and is recorded as skipped.
- Repair: round 1 untouched, round 2 touches a matching path → the gate runs in round 2.

## Acceptance criteria

- [ ] On adw-factory, a run whose diff doesn't touch the live paths spends **zero** Docker or E2B time in gates. Put the journal excerpt in the PR.
- [ ] A run touching `src/workspace/container.ts` runs `test-live`. Prove it with a test that drives the gates node with that diff.
- [ ] Other targets' gate behaviour is byte-identical: existing gates and baseline tests stay green with no edits.
- [ ] Spec amendment landed.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing). It touches `gates.ts` and `baseline.ts`. The baseline overlaps with adw-perf-07, which is in progress, and adw-bug-35. If either is in flight, rebase onto it rather than run alongside it.
