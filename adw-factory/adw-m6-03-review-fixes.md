---
id: adw-m6-03-review-fixes
type: chore
status: done
priority: 1
created: 2026-07-16
epic: adw-m6
depends: [adw-m6-02-metrics-tuning]
attempts: []
---
# M6 milestone external review — adversarial-review fixes

Operator ran `/codex:adversarial-review` over the M2+M3+M6 span
(base `d209e07`) 2026-07-16. Six findings, each challenged against code +
primary sources + ticket scope (m3-06 pattern). VALID → fixed red-first,
validator-gated; CHALLENGED → recorded here (operator may overrule at the
milestone review, treating overruled items as new in-epic work).

## Findings table

| # | Sev | Finding | Verdict |
|---|-----|---------|---------|
| 1 | high | agent allowlist exposes host credentials (build.ts) | **VALID (partial)** — env sanitized + printenv dropped; tool-restriction/OS-sandbox is M4 |
| 2 | high | CI adapter "crashes on failing checks" (ci-round/liveGh) | **VALID (reframed)** — failing-crash disproven by live evidence; pending (exit 8) real; fixed behavior-agnostic |
| 3 | high | ceilings advisory not hard (engine) | **VALID** — remaining-turn budget + wall-clock deadline signal |
| 4 | high | PR create not idempotent/crash-recoverable (open-pr) | **VALID** — reconcile via `gh pr list --head` |
| 5 | high | finalizeBlocked can overwrite operator edits (attempts) | **VALID** — dirty-ticket guard, throw not overwrite |
| 6 | med | capture accepts stale file after later append failure (clens) | **CHALLENGED** — re-raise of m3-06 recorded decision |

## Per-finding detail

### 1 — env exposure (VALID, partial) — src/live-query.ts, src/pipeline/nodes/build.ts
`liveQuery` copies ALL of `process.env` into the agent child; combined with
the `bun:*`/`bunx:*` auto-approve (which can run arbitrary JS) any operator
secret in the environment is reachable by prompt-injected/malicious ticket
content before the human gate. `printenv`/`printenv:*` in the allowlist is a
gratuitous exfil primitive (it was only ever needed for the one-time M6
TRACEPARENT probe).
- **Fix:** a pure `sanitizeAgentEnv(processEnv)` drops credential-bearing
  keys (`GITHUB_TOKEN`, `GH_TOKEN`, `ANTHROPIC_API_KEY`, and the
  `*_TOKEN`/`*_SECRET`/`*_KEY`/`*_PASSWORD`/`*_PASSWD` families) before the
  workspace env is merged over it; drop `printenv`/`printenv:*` from
  `AGENT_ALLOWED_TOOLS`.
- **Denylist not allowlist, deliberately:** v1 auth is the Claude login
  discovered via `HOME` (N1 — no `ANTHROPIC_API_KEY`), and gh via its
  keyring (also `HOME`). A strict allowlist risks silently starving the
  SDK's credential discovery (untestable without a live run); a
  pattern denylist removes the secrets with zero correctness risk. The
  strict allowlist + OS/network boundary is the **M4 container** boundary
  (`canUseTool`/sandbox) — explicitly deferred, not dropped.

### 2 — CI checks adapter (VALID, reframed) — src/cli.ts liveGh, src/pipeline/nodes/ci-round.ts
Codex framed this "no-ship: crashes on failing checks." **Disproven by live
evidence:** run `m2s-003-ci` executed `ci-repair` through the default
`liveGh` with no catch on the path — which only happens when `parseChecks`
returns `failing`, i.e. `gh pr checks --json` returned failing JSON WITHOUT
throwing. So the installed gh exits 0 on failing checks with `--json`.
The **real** exposure is `gh pr checks` **exit 8 (Checks pending)** — the
common state in the seconds after PR creation — which `execFileSync().trim()`
would throw on, aborting the pre-selection sync sweep before ticket
selection.
- **Fix (behavior-agnostic, so no gh-version archaeology):** `liveGh` uses
  `spawnSync` (returns `{status, stdout, stderr}`, never throws) and a pure
  `interpretGh(args, result)` classifier: exit 0 → stdout; a `pr checks`
  nonzero exit WITH non-empty stdout → return the stdout (the JSON rides
  regardless of exit 1/8); anything else nonzero → throw naming exit +
  stderr (auth/transport still fail loudly). Correct whether gh exits 0 or
  8 on any check state.

### 3 — hard ceilings (VALID) — src/pipeline/engine.ts
Two real gaps: (a) `boundaryBreach` only runs BETWEEN nodes, so a
long-running agent node has no wall-clock deadline; (b) every agent node
receives the FULL turn cap (`ctx.caps.turns`), and build's mid-stream
counter resets per stream — so a repair can spend another full allowance
before the next boundary sees the cumulative breach.
- **Fix (a):** an internal `AbortController` combining the external signal
  AND a wall-clock deadline (injectable timer seam, default
  setTimeout/clearTimeout); its signal is what ctx carries, so build's
  existing `ctx.signal.aborted` between-message check interrupts a
  slow-but-yielding node. A stream that never yields still needs OS-level
  timeout → M4 container (documented, not silently dropped).
- **Fix (b):** ctx.caps for each node carries `turns: caps.turns -
  turnsUsed` (remaining cumulative budget), making the turn ceiling hard
  across nodes.

### 4 — PR create idempotency (VALID) — src/pipeline/nodes/open-pr.ts
The bounded retry loop blindly re-runs `gh pr create`; if GitHub creates the
PR but the response is lost, the retry hits "a PR already exists for branch"
and the node fails, orphaning a live PR from factory state (sync follows
recorded in-review attempts, so the ticket can block despite an open PR).
- **Fix:** on create failure, reconcile via
  `gh pr list --head <attemptBranch> --state open --json url` — exactly one
  match is adopted as success (its url recorded); zero matches → the
  existing bounded-retry/fail path.

### 5 — finalizeBlocked dirty guard (VALID) — src/intake/attempts.ts
`finalizeBlocked` reads/rewrites/commits `tickets/<id>.md` without
re-checking cleanliness. Dispatch-time cleanliness (E1) is not durable
across a long run or a later CI invocation, so an operator edit racing the
finalize is silently swept into the factory commit or overwritten —
violating the dirty-ticket transition invariant (`open-pr.ts:323` already
guards this way).
- **Fix:** before mutating, `git status --porcelain -- tickets/<id>.md`;
  non-empty → throw descriptive (matching open-pr), never overwrite. The
  lesser evil in that rare race is a ticket left in-progress with a loud
  error + kept workspace, NOT clobbered operator edits.

### 6 — capture stale-file after later append failure (CHALLENGED) — src/observability/clens.ts
Re-raise of a decision the operator already recorded as CHALLENGED in
adw-m3-06 (durable-write-ack + crash-window WAL). The factory cannot obtain
a per-event durable acknowledgement from cLens: **cLens logs persistence
errors internally and exits 0 before its async write flushes**, so any
"verify the append offset/last JSONL record" check is racy — reading the
file immediately after exit 0 can false-negative (not yet flushed) or
false-pass (previous content). The review's "end-of-session completeness
check" is undefined without an expected event count. m3-06 already added
what IS soundly checkable factory-side (ok requires the session file to
exist; a later hook failure appends one `ok:false` downgrade;
last-record-per-session wins; journal-authoritative reconciliation). Durable
-write acknowledgement stays on the **cLens wishlist** (out of factory
scope). Rationale unchanged from m3-06; recorded for the operator to
overrule.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; each VALID fix
red-first and validator-approved; challenge rationale recorded.

## Outcome (2026-07-16)

All 5 VALID fixes landed red-first (validator-gated) and green; finding 6
CHALLENGE recorded. Suite 379 → **394** green, lint + tsc clean.
`/code-review` (two-axis) ran clean — **no hard standards violations, 5/5
fixes correctly implemented** — and surfaced two hardening follow-ups, both
applied: a final boundary check so a deadline firing during the TERMINAL
node still blocks (red-first), and threading the reconcile query's failure
cause into the E6 fail reason (Art. VI). `_PASSWD` added to the env denylist
families (code+ticket synced). Finding 6 remains open for the operator to
overrule at the milestone review.
