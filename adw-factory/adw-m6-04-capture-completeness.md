---
id: adw-m6-04-capture-completeness
type: chore
status: done
priority: 1
created: 2026-07-16
epic: adw-m6
depends: [adw-m6-03-review-fixes]
attempts: []
---
# End-of-run capture-completeness check (finding 6, overruled)

The operator overruled the adw-m6-03 **finding 6** CHALLENGE (capture accepts
a stale file after a later append failure) 2026-07-16 — "take the review
feedback and turn it into an applicable fix." Per adw-m6-03's own rule, an
overruled challenge is new in-epic work.

## What was challenged, and what IS soundly fixable

The unfixable part stands: the factory cannot get a **per-event durable
acknowledgement** from cLens (it exits 0 before its async write flushes), so
an inline post-exit-0 offset/last-record check is racy and stays out of
scope (cLens wishlist).

But the review's **second** recommendation — an end-of-session completeness
check — IS soundly implementable at ONE point: **run end**. By the time
`runLane` returns, every `clens-hook` subprocess spawned during the agent
session has exited (they are spawned synchronously per event), so reading the
session files then is race-free — far from any write.

**Scope, stated honestly (this does NOT "fix finding 6"):** finding 6's exact
scenario is a later append that *fully fails* while cLens still exits 0,
leaving a **valid but shorter** file — every line parses, so this check
passes it. Detecting that needs an expected event count the factory does not
have; it remains a cLens-side durable-write-ack limitation (out of factory
scope, cLens wishlist) — the same conclusion the original challenge reached.
Since cLens appends **whole lines** (verified on a real 317-line artifact),
on real output this check fires only on genuine **mid-write truncation or
corruption** (a crash) or an empty file — narrower, defensive integrity
adjacent to the finding, not its core. Shipped because it is sound, cheap,
and strictly strengthens the "answerable from artifacts" guarantee; not
oversold.

## Requirements

- **Pure** `validateCaptureContent(content: string)` in
  `src/observability/clens.ts`, exported: `{ ok: true }` when the content is
  non-empty (trimmed) AND every non-empty line parses as JSON; else
  `{ ok: false, error }` naming the failure (empty / malformed line N /
  truncated final record). No I/O, no clock (Art. IX). Red-first.
- **Edge** `checkCaptureCompleteness(runDir, append)` (hand-verified I/O
  edge, like `liveClensSpawn`): enumerate `runDir/.clens/sessions/*.jsonl`
  (excluding `_links.jsonl`), validate each, and for any that fail append a
  `capture` downgrade record `{ ok: false, sessionId, path, error }` — reusing
  the existing last-record-per-session reader rule (adw-m3-06 F1), so
  `renderSessionIds` and metric 3 flag it. Missing `.clens` dir (capture
  disabled) → no-op. Never throws (the capture-tolerance stance, m3-01/m3-04).
- **Wire** it in `runTicket` (`src/cli.ts`) immediately after `runLane`
  returns, before the green/blocked branch, so it runs on EVERY outcome
  (metric 3 covers blocked + aborted runs too).

## Contracts (validator-required, recorded here)

- **Coverage:** the check runs after BOTH the main lane (`runTicket`) AND the
  CI mini-lane (`ciRound`, against the prior run's shared `.clens` sink) — a
  truncated ci-repair transcript is flagged too, not just the build session.
- **Journal ordering:** the downgrade record lands AFTER the lane's `run-end`
  (the check runs post-`runLane`). It is a `capture` record, so the
  last-record-per-session reader rule (m3-06 F1) still resolves it correctly;
  but any metric-3 reader MUST scan the whole journal, not stop at the first
  `run-end`. `status.ts` (scans backward for run-end) and `open-pr`'s
  `renderSessionIds` (`.find`) are already robust.
- **PR-body timing (documented limitation):** `renderSessionIds` runs
  mid-run inside `open-pr`, BEFORE this end-of-run check, so a GREEN run's PR
  body reflects only per-event capture health. The authoritative end-of-run
  structural verdict lives in the journal (`adw status` / metric 3), not the
  PR body. Closing that gap (running the check pre-open-pr for green runs)
  was judged disproportionate for v1 — the operator sees truncation via the
  run artifacts, which is the finding's core "answerable from artifacts"
  guarantee.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; the pure validator
red-first and validator-approved; `/code-review` clean.

## Outcome (2026-07-16)

Pure `validateCaptureContent` (red-first, validator-approved) +
`checkCaptureCompleteness` edge — unit-tested directly (readdir+readFile+append,
not a real subprocess; the truncated→downgrade branch is exercised), wired
after `runLane` in both `runTicket` and `ciRound`. Suite 394 → **401** green,
lint + tsc clean. Sound by
construction (run-end read is race-free — all clens-hook subprocesses have
exited) and insensitive to benign flush-lag (a clean lag drops whole valid
lines → still ok; only truncation/corruption/emptiness fails). Residual
out-of-scope items unchanged: per-event durable-write ack (cLens wishlist),
silently-dropped-event detection (needs an expected count).
