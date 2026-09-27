---
id: adw-m7-03-codex-ci-auth-provenance
type: feat
status: done
priority: 3
created: 2026-07-19
epic: adw-m7
depends: [adw-m7-01-codex-worktree]
attempts: []
---
# Attempt-aware CI auth + resume-model provenance (Codex)

> Minted 2026-07-19 from adw-m7-01's round-2 adversarial review
> (`ai_docs/2026-07-19-adw-m7-01-codex-adversarial-review-round2.md`). Coarse —
> refine at pickup. Splits the mixed-provider CI concerns out of m7-01 so the
> fresh-run worktree path can land.

## Scope

Two deferred round-2 findings, both about maintaining an EXISTING PR whose
originating provider differs from the current invocation's:

1. **#2 [high] CI auth preflight is scoped to the current provider, not the
   attempt.** `checkCodexAuth` runs only when the current flag/target resolves
   to Codex, before sync — but sync routes by the ORIGINATING attempt. So a
   Claude invocation can reach a Codex PR with no binary/auth check, after which
   `ciRound` charges its sole retry before spawning Codex (→ wasted round,
   blocked). Conversely, an unauthenticated Codex-selected invocation refuses
   before reconciling a legacy Claude / already-merged / closed PR. **Fix
   direction:** keep fresh-dispatch auth separate from synchronization; pass an
   auth checker into `ciRound`; validate a Codex attempt's auth+binary BEFORE
   the charge or touching its workspace.

2. **#6 [medium] Mutable ticket.model overrides persisted attempt.model on
   resume (PARKED at m7-01).** CI resolves `ticket.model` before
   `attempt.model`, so editing the ticket after PR creation silently resumes the
   Codex session on a different model (the exact mismatch the resume fixture
   warns about) and may spend the charged round on an incompatible model.
   **Candidate resolution (operator to confirm):** fresh honors `ticket.model`
   (S6.3); resume uses the attempt's PERSISTED model; a model change forces a
   re-provision (fresh session), not a resume — a clarification to S6.3.

## Out of scope

The fresh-run provider swap (adw-m7-01); container/e2b Codex parity; capture
(adw-m7-02).

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; a failing-CI test for a
Codex-created PR met by a Claude invocation (auth checked before charge); a
pre-provenance Claude attempt still repairs on Claude.

## Status — 2026-07-19 (built, offline TDD; validator-gated)

**#2 attempt-aware CI auth — DONE (both halves).**
- (a) `ciRound` validates codex auth by the ORIGINATING attempt's provider
  BEFORE the charge; missing auth → `action:"skipped"` (RECOVERABLE — ticket
  stays in-review, no charge, relaunch), mirroring the container/e2b
  skip-before-charge precedent (NOT `blockedNoCharge`, reserved for
  unrecoverable state errors). A codex attempt with no auth-checker wired
  throws (Art. IX). Checker bound in `main()` over `codexAuthPreflight`,
  threaded into `ciRound` deps independent of the run's own `--provider`.
- (b) the fresh-dispatch codex-auth + `codexQuery`-binding refusals MOVED to
  AFTER `syncPrState` (before ticket selection): sync runs unconditionally
  (S3.2) — an unauthenticated `--provider codex` run can now reconcile a
  merged/closed/legacy PR — and codex auth gates only a FRESH dispatch. The
  unsupported-REQUEST refusals (provider recognition + codex/non-worktree
  isolation) stay before sync.

**#6 resume-model provenance — part A DONE, part B DEFERRED (operator, doc for
later pickup).**
- Part A: `ciRound` resume resolves `attempt.model ?? providerDefault` — pins
  to the PERSISTED attempt model, no longer lets a mutable `ticket.model` edit
  override on resume. Fresh dispatch (chore lane, S6.3) untouched.
- **Part B — PARKED for a later pickup:** honoring a `ticket.model` ≠
  `attempt.model` edit on a codex CI resume. The silent-drift HARM is already
  gone (part A); this is the "how to handle a deliberate mid-flight model
  change" question. Three candidate resolutions to weigh at pickup:
  (1) leave as-is — resume pins to attempt.model; to switch models,
  reject/requeue → fresh dispatch honors `ticket.model` (S6.3); document only.
  (2) mismatch guard — on `ticket.model` ≠ `attempt.model`, SKIP with a message
  telling the operator to requeue (surfaces intent instead of ignoring it).
  (3) full re-provision — a model change starts a FRESH codex session (no
  resume) on `ticket.model` in the kept worktree; needs an S6.3 spec amendment
  (stop-and-propose per the amendment rule).
