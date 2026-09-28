---
id: adw-learn-04-a-blocked-run-says-why-in-one-word-21791c
type: feat
status: blocked
priority: 2
created: 2026-09-28
review: false
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-learn-04-a-blocked-run-says-why-in-one-word-21791c-1790621544351","branch":"adw/adw-learn-04-a-blocked-run-says-why-in-one-word-21791c","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-learn-04-a-blocked-run-says-why-in-one-word-21791c-1790621544351/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# A blocked run says why in one word, before it says why in a sentence

> `adw-learn-*` group (`ai_docs/2026-09-28-prompt-training-readiness.md` §3.4).
> Zero-token plumbing. `review: false`: hard-gated.

## The defect this closes

`run-end.reason` is free text (`journal.ts:595`), assembled at eleven sites
in `engine.ts` (`:541-833`) and by every node's `fail()`. Measured 2026-09-28
over 78 blocked runs, the reasons fall into about eight families, but
recovering them takes a regex over prose such as:

```
ticket "…" node "dispatch": status dispatch for "…" …
ticket "…": node "gates" exhausted 3 repair rounds
turn ceiling breached: 646 turns used, cap 600 — refused agent node "review-fix"
ticket "…" node "build": no-progress detector …
ticket "…" node "build": agent reported an error result: API Error: …
```

`ai_docs/2026-09-27-supervisor-toolbox.md` §B says the same from the other
side: *"Free text for the deadline and the livelock stop. **Nothing** records
repair-round exhaustion."* A failure taxonomy is the first thing any learning
loop groups by, and today it is a `sed` script.

## Requirements

- [ ] **R1** A closed union `BlockedCause` in `journal.ts`:
      `"dispatch-refused" | "provision-failed" | "ceiling-turns" |
      "ceiling-wall" | "ceiling-context" | "watchdog-stall" | "no-progress" |
      "gates-exhausted" | "red-check-exhausted" | "review-exhausted" |
      "agent-error" | "tool-denied" | "node-fail" | "aborted" | "thrown"`.
      Adding a member is a type change, so a new cause can never be journaled
      as prose by accident.
- [ ] **R2** `run-end` gains `cause?: BlockedCause` beside `reason`. Every one
      of the eleven producing sites in `engine.ts` sets it; `reason` stays
      exactly as today so the CLI line and `just fails` are byte-identical.
      A node's `fail()` maps to `node-fail` unless the node supplies a more
      specific cause through `NodeResult`.
- [ ] **R3** The **exhausted-rounds** case, unrecorded today, journals its
      own `round` event with `exhausted:true` at the ceiling, naming the loop
      (`gates↻repair`, `red-check↻revise`, `review↻fix`) and the round count.
- [ ] **R4** `finalizeBlocked` writes `cause` into the ticket's `attempts[]`
      entry beside `outcome`, so the ledger can be grouped without the journal.
- [ ] **R5** `scripts/run-metrics.ts` prints a `blocked by cause` table; the
      web board's failure chip reads `cause` first and falls back to today's
      regex on `reason` for unstamped journals.
- [ ] **R6** A **backfill classifier**, `scripts/backfill-cause.ts`: pure
      function `reason → BlockedCause | undefined` with one banked real
      `reason` string per family as its fixture, reporting how many of the
      current 78 it could classify. It writes nothing; it exists so the
      historical corpus can be labelled once, offline, and so the mapping is
      tested against what the engine really wrote.

## Verify

- [ ] Red first (Art. I).
- [ ] Each `BlockedCause` member is produced by at least one engine test.
- [ ] A run blocked at three repair rounds journals `cause:"gates-exhausted"`
      and a final `round` event with `exhausted:true`.
- [ ] The backfill classifier labels every one of the 78 banked reasons or
      names the ones it cannot.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Out of scope

Changing what any bound does when it trips (Art. V). Reworking the CLI's
exit codes (`1` still means blocked and crashed both; that is the
supervisor's ticket).
