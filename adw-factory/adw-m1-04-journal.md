---
id: adw-m1-04-journal
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends: [adw-m0-01-toolchain]
attempts: []
---
# Run journal — JSONL

## Context

Layer 1 of the observability trio. "A run that cannot be replayed from its
artifacts is a defective run" (Art. VI). The journal is the only run history
(Art. VIII); M2's PR body stats are computed from it.

## Deliverables

- `test/observability/journal.test.ts` (first, red)
- `src/observability/journal.ts`

## Requirements

- [x] One file per run: `runs/<runId>/journal.jsonl` (gitignored); append-only,
      one JSON object per line, each line self-contained and carrying
      `runId` + `ticketId` + timestamp (S5.1)
- [x] Event types (typed union): `run-start` (isolation kind), `node-start`,
      `node-end` (outcome + details: gate results, or agent usage
      `{tokens, turns, sessionId}`), `round` (`{node, round, of}`), `abort`
      (`{reason}`), `run-end` (`{outcome, durationMs}`) (S5.1)
- [x] Crash-safe: each event flushed on append — the file is valid JSONL
      after abnormal termination at any point (S2.6 "artifacts flushed")
- [x] `readJournal(runId)` parses a journal back into typed events (consumed
      by `adw status` and the PR body in M2)

## Build protocol (Art. I)

1. Tests: event shape round-trips; append-then-read equality; partial-file
   readability (simulate truncation after N lines); ticketId on every line.
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

OTel spans and exporters (M3), cLens capture (M3).
