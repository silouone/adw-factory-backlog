---
id: adw-m3-06-review-fixes
type: feat
status: done
priority: 2
created: 2026-07-15
epic: adw-m3
depends: [adw-m3-05-replay-audit]
attempts: []
---
# External-review fixes (Codex adversarial review of M3)

Operator-directed 2026-07-15: challenge the Codex findings; the survivors
become red-first fixes inside epic adw-m3 before M6 starts. Verdicts below
were checked against the code, the cLens source, and the tickets' declared
scope — each challenge is recorded so the milestone review can overrule.

## Findings & verdicts

| # | Codex finding | Verdict | Action |
|---|---------------|---------|--------|
| F1 | capture `ok:true` from FIRST hook result only, deduped forever; cLens exit 0 isn't a durable-write ack (its hook logs persistence errors and exits 0 — verified) | **VALID** | track status across every event: a later failure journals a DOWNGRADE `capture` record (`ok:false`); `ok` additionally requires the session file to exist; readers take the LAST record per session; PR body flags incomplete captures |
| F2 | CI repair appends to the PRIOR run's session JSONL | **CHALLENGED** — cLens resolves its sink from the hook payload's `cwd` (the kept workspace) and modifying cLens is a spec non-goal; the resumed conversation IS one session, its file the session's artifact, explicitly referenced by both runs' journals; ci-round already hard-blocks when the prior dir is gone | document in the m3-05 runbook; "capture root override" filed on the cLens wishlist (M6) |
| F3 | `Bun.spawnSync` without timeout: a wedged clens-hook blocks the event loop and bypasses ceilings | **VALID** | 5s timeout on `liveClensSpawn` (Bun supports `timeout`); per-session circuit breaker — after a failed capture, later events for that session skip the spawn (status already `ok:false`, bounded stall) |
| F4 | TRACEPARENT proof is syntactic, not live cross-process | **CHALLENGED** — adw-m3-03 declares "consuming TRACEPARENT inside agent processes" out of v1 scope; live agent-env receipt is an M6 item (agents have no Bash under acceptEdits) | uplift anyway: a REAL child-process test (OS boundary, zero tokens): spawn `bun -e`, child reads env, extracts with the W3C propagator, exports a child span; parent asserts traceId + parent span id |
| F5 | kill switch swaps only the exporter — spans + TRACEPARENT still flow when disabled | **VALID** | `tracingDisabled(env)` predicate; the CLI omits the tracer AND the ciRound tracing factory entirely when disabled; test pins NO `TRACEPARENT` in agent env when disabled |
| F6 | span.end before journal node-end isn't crash-consistent (microsecond window) | **CHALLENGED** — a WAL/completion protocol is disproportionate for v1 (Art. VIII); either ordering leaves a one-sided window | document the reconciliation rule in the runbook: the JOURNAL is authoritative; an ended span whose node lacks `node-end` = death in the boundary window |
| F7 | span files written non-atomically to final names | **VALID** (cheap) | write `.tmp-` sibling + `renameSync` (atomic on same fs); runbook: readers consider only `\d{6}\.json` |
| F8 | ticket tag not persisted IN the session JSONL (cLens ignores `ADW_TICKET_ID` — verified, zero hits in its source) | **VALID** | enrich the forwarded payload with `adw_ticket_id` — cLens persists the payload verbatim in `full` capture mode (verified hook.ts `data = input`); journal join stays the authoritative mapping (Art. VIII); metadata-mode caveat documented |
| F9 | capture dirs/files world-readable (0755/0644 observed live) | **VALID** (minimal) | pre-seed `runs/<runId>/.clens` chain at `0700` (mkdir mode + explicit chmod); broader run-dir policy + cLens file modes → M6 |

## Validator round (red gate) — REVISE applied

1. **Blocking gap fixed:** the cli-level capture test's fake spawn now
   simulates cLens's durable append (writes the session JSONL), so F1's
   ok-requires-file semantics keep the whole suite consistent at green.
2. **F5 ci-round half — recorded justification:** the disabled gate lives in
   ONE decision point in cli.ts (`tracingFor`), shared verbatim by
   `runSelected` and the `ciRound` binding; the main-run disabled test pins
   that gate's observable contract (no TRACEPARENT, no spans dir). ciRound's
   absent-factory behavior (untraced, zero change) is already pinned in
   ci-round.test.ts Parts A–E/F. A cli-level CI-round disabled test would
   need the full in-review+failing-checks fixture for no additional
   discrimination — deferred with this note per the gate's own bar.
3. **F3 nudge acknowledged:** `liveClensSpawn`'s 5s timeout is the untested
   production edge (same stance as liveQuery); the breaker algorithm is the
   tested half. The timeout is REQUIRED in the implementation — reviewer
   checks it at green.

## Verify

Red tests first (validator gate per protocol), then green:
`bun run lint && bunx tsc --noEmit && bun test` — all green; runbook
amendments recorded in tickets/adw-m3-05-replay-audit.md.

## Out of scope

Modifying cLens (spec non-goal — wishlist items recorded for its own
backlog: durable-write ack, capture-root override, file modes); WAL/crash
protocol (F6); live agent-env TRACEPARENT receipt (M6, permission-mode
decision).
