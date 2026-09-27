---
id: adw-gates-07-a-target-declares-how-its-runner-reports-failures-ad75a0
type: feat
status: in-review
priority: 2
created: 2026-09-27
review: false
caps: {minutes: 150, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-gates-07-a-target-declares-how-its-runner-reports-failures-ad75a0-1790545950932","branch":"adw/adw-gates-07-a-target-declares-how-its-runner-reports-failures-ad75a0","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-07-a-target-declares-how-its-runner-reports-failures-ad75a0-1790545950932/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/138","provider":"claude","model":"sonnet"}]
---
# A target declares where its runner writes a machine-readable report, so the factory stops scraping reporter prose

> Operator decision 2026-09-27, after `cqc-fe-01-drawer-keeps-keyboard-focus`
> blocked twice: *"we can't have to adapt the factory for each new
> language/test framework."* This ticket moves per-runner knowledge out of
> factory regexes and into target **config**, where `cmd`, `scopedCmd` and
> `allowedTools` already live. `review: false`: hard-gated.

## The measurement that decides this

Across every run in `runs/`, here is every non-empty `failingTests` ever
journaled:

| runner | non-empty entries |
|---|---|
| pytest (`tests/*.py::test_*`) | **9** |
| jest | **0** |
| bun | **0** |

…plus **16** empty ones. `parseFailingTests` is fed `exec.stdout`
(`src/pipeline/nodes/gates.ts:459`) while `extractNotices` is correctly fed
`exec.stderr` (`:450`). **pytest works only because it is the one of the three
that writes its summary to stdout.** Verified locally 2026-09-27:

```
$ bun test x.test.ts 2>&1 >/dev/null
(fail) deliberately failing [0.13ms]        # matches BUN_FAIL_LINE — on stderr
```

So `parseBunFailingTests` has returned `[]` since the day it was written,
including on adw-factory's own self-hosted gate. And the fix attempts keep
shipping green because each one's fixture encodes the same wrong premise:
`adw-gates-05`'s own test passes jest reporter text as
`execResult({ code: 1, stdout })`, and its R1 says *"parse jest's default
reporter **stdout**"*. Same fixture-provenance defect `review-spec` flagged on
`adw-pr-02` ("bank a real payload as the fixture"). A synthesized fixture
confirms the assumption instead of testing it.

One regex per framework (gates-03 pytest, gates-05 jest, a gates-0N for vitest…)
is a treadmill, and each rung is a stdout-scrape of prose the runner never
promised to keep stable.

## What replaces it

Every runner already emits a machine-readable report natively, **no new
dependency**. Verified 2026-09-27:

- **bun** — `bun test --reporter=junit --reporter-outfile=<path>`:
  ```xml
  <testcase name="deliberately failing" classname="" time="0.000297"
            file="x.test.ts" line="2" assertions="1">
    <failure type="AssertionError" message="Expected: 2&#10;Received: 1">
  ```
  Names, file, line **and** the assertion message — richer than the regex.
- **pytest** — `--junitxml=<path>`, built in.
- **jest 24** (`cmc`) — has **no** bundled JUnit reporter (`jest-junit` is a
  third-party dep, and `cmc` forbids new deps and editing `package.json`), but
  `--json --outputFile=<path>` is built in, and `cmc`'s CRA wrapper ends in
  `jest.run(argv)` (`scripts/test.js:32`), so
  `npm test -- --json --outputFile=…` passes straight through with **no repo
  edit**. Its shape (`testResults[].assertionResults[].fullName` + `status`) is
  quoted from documentation, **not measured** — confirm it against a real run
  before relying on it (R4 requires a banked payload anyway).

Two readers, both stable documented schemas, cover essentially every runner the
factory will ever meet. The per-runner *flag* lives in each target's own `cmd`.

## Requirements

- [ ] **R1** `Gate` (`src/targets/loader.ts:38-46`) gains optional
      `report?: { path: string; format: "junit" | "jest-json" }`. `report` joins
      `KNOWN_FIELDS`; `path` a non-empty **relative** path, `format` one of the
      two literals; anything else is a reported validation error naming the
      field (precedent: `allowedTools`, adw-tools-02 R1). Absent → not present
      on the `Gate`.
- [ ] **R2** **No new templating.** The target writes the literal output path
      into its own `cmd`; the factory only needs to know where to read it.
      `{{files}}` substitution lives in `red-check.ts:275`, outside this
      ticket's surface — do **not** add a `{{report}}` placeholder.
- [ ] **R3** `gates.ts`: **delete `report.path` before running** a gate that
      declares one, so a stale report from a previous repair round can never be
      read as this round's result. After a non-zero exit, if the file exists,
      parse it and use its ids as `failingTests`.
- [ ] **R4** Two **pure** readers:
      `junit` → one id per `<testcase>` carrying a `<failure>` or `<error>`
      child, as `<testsuite name>::<testcase name>`;
      `jest-json` → `testResults[].assertionResults[]` where
      `status === "failed"`, id `fullName`.
      Each is tested against a **banked real payload captured from the actual
      runner** — not hand-written. This is the specific defect that let
      gates-03 and gates-05 ship green, so a synthesized fixture does not
      satisfy this requirement.
- [ ] **R5** A declared report that is **missing or unparseable** after a
      failing exec is a **notice naming the gate and the path**, never an error
      and never a silently empty list — then fall back to R6. Same tolerant-read
      stance `loadRuns` and adw-gates-01's loud-skip notices already take.
- [ ] **R6** The three stdout regexes (`BUN_FAIL_LINE`, `PYTEST_FAIL_LINE`,
      `JEST_FAIL_LINE`/`JEST_SUITE_LINE`) **stay as the fallback** for targets
      that declare no `report` — and their one-line stream bug is fixed:
      `parseFailingTests` is fed stdout **and** stderr. `adw-gates-05`'s
      existing tests stay green untouched; add one test proving the stderr case,
      against a banked real payload (R4's rule applies here too).
- [ ] **R7** Declare `report` for the **three verified** targets only:
      `adw-factory` + `clens` (bun → `junit`), `sabado` (pytest → `junit`),
      `cmc` (jest → `jest-json`). The path lives under `.adw/` — the factory's
      own per-workspace scratch space — which push **already** keeps out of the
      commit: `git add -A -- .` is followed by
      `UNSTAGE_SCRATCH_CMD = "git reset -q -- .claude .adw"`
      (`src/pipeline/nodes/push.ts:125-130,158`), whose own comment names `cmc`
      as a target that does not gitignore `.adw/`. So no `.gitignore` edit is
      needed and `cmc`'s standing overlay rule (*never modify tracked files*) is
      respected — but **assert it**: a test or verify step proving the report
      never reaches the pushed diff. The other eight targets keep the R6
      fallback.
- [ ] **R8** `.claude/skills/adwf/references/targets.md` documents the field,
      both formats, and the flag for each runner (bun / pytest / jest, plus the
      `go test -json` and `cargo nextest --profile ci` paths for the next
      target that needs one).

## Files this ticket owns — parallel-safety

`adw-gates-06-an-unparsed-runner-cannot-trip-the-livelock-check-2c9572` is
fired **concurrently** with this ticket. They are disjoint by construction:

- **This ticket touches** `src/pipeline/nodes/gates.ts`, `src/targets/loader.ts`,
  `targets/*.json`, `.claude/skills/adwf/references/targets.md`, and tests.
- It must **not** touch `gateFingerprint` or `src/pipeline/engine.ts` — gates-06
  owns the livelock check, and its digest is deliberately independent of whether
  names are populated, so nothing here needs to reach into it.
- It must **not** add a field to `GateResult` (`src/observability/journal.ts`).
  `failingTests` already exists; this ticket only changes where its contents
  come from.

If implementation seems to require touching a gates-06 file: **stop and propose
a spec amendment** (project CLAUDE.md) rather than reaching across the fence.

## Verify

- [ ] Red first (Art. I) — tests written, confirmed red, reviewed, then green.
- [ ] A target declaring `report` whose suite fails → `failingTests` names every
      failure, sourced from the report file, with **zero** stdout/stderr
      scraping on that path.
- [ ] A declared report the runner never wrote → notice + R6 fallback; the run
      is not failed and not silently nameless.
- [ ] A stale report left by the previous round is never read (R3).
- [ ] Both readers green against a banked real payload from the actual runner.
- [ ] A target with no `report` behaves exactly as today, except that a bun or
      jest failure now names its tests (R6's stream fix).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.

## Out of scope

The livelock fingerprint (`adw-gates-06-…-2c9572`). Declaring `report` for the
other eight targets, and **removing the R6 fallback once they all do** — that
removal is a deliberate follow-up ticket, not an excuse to leave the fallback
permanent. Per-test identity on a red baseline beyond what `failingTests`
already feeds `classifyGateOutcome`.
