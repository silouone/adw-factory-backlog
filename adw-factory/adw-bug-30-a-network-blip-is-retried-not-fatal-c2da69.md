---
id: adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69
type: bug
status: in-progress
priority: 1
created: 2026-09-28
caps: {minutes: 180, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69-1790621511343","branch":"adw/adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69-1790621511343/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69-1790690661501","branch":"adw/adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-30-a-network-blip-is-retried-not-fatal-c2da69-1790690661501/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/164","provider":"claude","model":"claude-sonnet-5-5","rebased":"9a528dcb37edb88e86d56f9a88cd3bada2a2d940"}]
---
# A 13-minute network blip blocked 10 of 11 in-flight runs; the transient retry recognised none of the errors

> **Evidence, 2026-09-28, 18:36–18:49 CEST.** A local network outage. Codex's
> own log (`~/.codex/logs_2.sqlite`) shows `account/rateLimits/read timed out`
> from 18:35:51 to 18:40:39. Ten runs ended `blocked` and one survived
> (`adw-board-04…-1790610034294`, green at 18:52). Nothing was wrong with the
> code in any of the ten.
>
> | run-end (CEST) | run | provider | node | `reason` (verbatim tail) |
> |---|---|---|---|---|
> | 18:36:35 | `adw-board-05-…-1790610062532` | claude | review-spec | `API Error: Connection closed mid-response. The response above may be incomplete.` |
> | 18:36:35 | `adw-pr-02-…-1790610114317` | claude | build | `API Error: Connection closed mid-response. …` |
> | 18:40:34–40 | `cqc-fe-10`, `cqc-fe-12`, `cqc-fe-07`, `cqc-fe-08`, `cqc-be-02`, `cqc-be-03` (6 runs in 6s) | codex (gpt-5.6-sol) | plan / build / test / review-fix | `Reconnecting... 2/5 (request timed out)` |
> | 18:45:39 | `adw-card-02-…-1790609830123` | claude | review-standards | `API Error: Unable to connect to API (UNKNOWN_CERTIFICATE_VERIFICATION_ERROR)` |
> | 18:49:21 | `adw-graph-02-…-1790611118960` | claude | plan | `API Error: Connection closed mid-response. …` |
>
> Six of the ten were salvaged by hand into PRs (~4,900 lines of work in their
> workspaces). The other four are being restarted: three had only a plan, and
> `adw-pr-02` died in `build` with only two red test files.

## Two defects, both needed for this to happen

### 1. The transient classifier knows one string

`src/pipeline/nodes/build.ts` (`TRANSIENT_RESULT_PATTERN`):

```ts
const TRANSIENT_RESULT_PATTERN = /stalled mid-stream/i;
```

adw-resilience-01 deliberately keyed it on the one message it had evidence
for. This ticket brings the evidence for three more. None of them match, so
`retryTransient` (plan) and `retryTransientIfClean` (build, test) never fired,
even for `cqc-fe-10`/`cqc-fe-12`/`adw-graph-02`, which died in `plan`, the one
node where a retry is always safe.

### 2. The Codex adapter ends the run on a progress notice

`src/codex-query.ts`, `case "error":` treats a top-level `error` event as the
terminal result. Its own comment flags this as unverified:

> ASSUMPTION: treated as terminal/is_error like turn.failed … If a real stream
> ever emits this NON-terminally the loop would end early; unverifiable
> without a live occurrence — flagged for the live bar.

This is the live occurrence. `Reconnecting... 2/5` means Codex was on retry 2
of 5 and still running. The adapter ended the stream there, and all six Codex
runs died with three retries left. **Unconfirmed:** that the message arrived
as a top-level `error` event rather than `turn.failed` or an `item`. No raw
Codex stream was kept for these runs. The text is the evidence (only the
`error` and `turn.failed` branches return a message verbatim, and a terminal
`turn.failed` would not say "2/5"), but R0 must settle it.

### Also in scope: the review nodes never retry

`adw-board-05` died in `review-spec` and `adw-card-02` in `review-standards`.
Reviews are read-only, so a restart of a review node is as safe as `plan`'s.
Yet neither review node opts into `retryTransient` (`feat.ts` / `bug.ts` /
`shared.ts`: only plan, build and test do).

## Requirements

- [ ] **R0 — capture before you classify.** Get one real Codex `--json`
  stream that contains a reconnect. Either reproduce it (e.g. `codex exec
  --json` with the network cut mid-turn), or show from the pinned Codex CLI
  source (0.154.0) which event type carries `Reconnecting... N/M`. Commit it
  as a fixture under `test/fixtures/codex/`. R2's shape follows the fixture,
  not this ticket's guess.
- [ ] **R1 — widen the Claude-side classifier, verbatim.** `isTransientResultError`
  also matches `Connection closed mid-response` and
  `Unable to connect to API (UNKNOWN_CERTIFICATE_VERIFICATION_ERROR)`, taken
  from the table above. Each gets a red test with the exact journal string. A
  real task failure (e.g. `API Error: 400 …`, the poisoned-session shape) must
  still NOT match.
- [ ] **R2 — a Codex reconnect is progress, not a result.** A `Reconnecting...
  N/M` notice (in whatever shape R0 found) bumps liveness and ends nothing.
  The stream carries on to `turn.completed` → success, or `turn.failed` →
  error, as before. If Codex gives up (the final `N/N` followed by
  `turn.failed`, or the stream closes), the resulting error text is itself
  transient-classified, so R1's retry applies. Red test: fixture stream
  `thread.started → … → error(Reconnecting 2/5) → … → turn.completed` yields
  one success result, not an error.
- [ ] **R3 — review nodes retry transients.** `review-standards`,
  `review-spec` and `review-fix` opt into `retryTransient`, or
  `retryTransientIfClean` for `review-fix`, which writes. Use the existing
  bounded-rounds machinery. Red test in the lane suite: a transient error
  result on `review-spec` produces a `round` event and a second attempt, not
  `run-end blocked`.
- [ ] **R4 — journaled.** Every transient retry already emits a `round` event.
  Confirm the new classes appear there with the verbatim `reason`, so the
  next outage is countable from journals (Art. VI).

## Out of scope

Continuing a run that is already `blocked`: that's
`adw-resume-01-continue-a-blocked-run-9ae0cb`. This ticket only stops a blip
from blocking in the first place.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`, green.
- The R0 fixture exists and R2's test replays it.
- `grep -c 'stalled mid-stream\|Connection closed mid-response\|UNKNOWN_CERTIFICATE_VERIFICATION_ERROR' src/pipeline/nodes/build.ts` ≥ 3.
