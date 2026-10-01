---
id: adw-bug-34-a-gate-report-directory-is-assumed-to-exist-7c1e05
type: bug
status: done
priority: 1
created: 2026-10-01
caps: {minutes: 60, turns: 120, stallMinutes: 15}
depends: []
attempts: []
---
# A gate's report directory is assumed to exist, so a green base reads as red

Measured 2026-10-01 on `cqc-fe-29` and `cqc-fe-32` (target `cmc`). Both runs ended
`blocked` at `baseline-green-check` with:

> gate "test" is already failing on the base commit ee2d1a176978… (the failing test
> names could not be read)

The base commit was green. Running the gate's own command by hand in the run worktree:

```
Test Suites: 123 passed, 123 total
Tests:       1073 passed, 1073 total
Error: ENOENT: no such file or directory, open '/app/.adw/report.json'
npm ERR! Test failed.
```

Every target writes its gate report under `.adw/` — a directory no target commits, so a
freshly cut worktree does not have it. jest 24 does **not** create the parent of
`--outputFile`: it reports every test green and then exits 1 on ENOENT. The baseline reads
exit 1 as a red base and refuses the run before a token is spent (S2.7). The report is
never written either, which is why the failing test names "could not be read" — the
`unattributable: true` in the journal is the same cause, not a second bug.

`clearGateReport` (`src/pipeline/nodes/gates.ts`, the one implementation shared by the
`gates` and `baseline` nodes since adw-bug-31) did `rm -f -- "<path>"` and nothing else.
`rm -f` on a path inside a missing directory succeeds silently, so the gap was invisible.

Exposure: every target declaring a report — `cmc` (`.adw/report.json`), `clens` and
`adw-factory` (`.adw/report.xml`), `sabado` (`backend/.adw/report.xml`). `adw-factory`
commits `.adw/artifacts/.gitkeep`, so its own worktrees were never exposed; `cmc` broke
first because jest is the runner that hard-fails on the missing parent.

## The fix

`clearGateReport` creates the report's directory in the same breath as deleting the stale
report: `mkdir -p -- "<dirname>" && rm -f -- "<path>"`, as one exec so R3's
"deleted BEFORE the gate command runs" guarantee is unchanged. A report at the worktree
root (dirname `.`) still runs the bare `rm -f`.

## Verify

- `clearGateReport` emits the `mkdir -p … && rm -f …` form for a nested report path, the
  bare `rm -f` for a root-level one, and nothing for a gate declaring no report.
- The cmc `test` gate on `ee2d1a1`, with `.adw/` absent, exits 0 and writes
  `.adw/report.json` with `success: true`.

Built in-session 2026-10-01 (operator-reported board blockage, red test first per Art. I).
