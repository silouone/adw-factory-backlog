---
id: adw-m5-06-remote-capture-parity
type: feat
status: done
priority: 1
created: 2026-07-19
epic: adw-m5
depends: [adw-m5-02-e2b-workspace]
attempts: [{"runId":"adw-m5-06-remote-capture-parity-1789085359632","branch":"adw/adw-m5-06-remote-capture-parity","workspace":"/Users/silouane/adw-factory/runs/adw-m5-06-remote-capture-parity-1789085359632/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# cLens capture parity for the remote (E2B) lane

> Minted 2026-07-19 during the adw-m5-03 refinement (operator-approved: defer
> the async-fetchTranscript rework out of the security-touching CI-round
> ticket into this cross-cutting one). The gap is an EXPLICIT deferral
> recorded since adw-m5-02, not a hidden hole — remote runs journal `capture
> ok:false` today and the CLI/journals say so; this ticket closes it for BOTH
> remote lanes at once (main + CI mini-lane).

## Context

The container kind reached cLens capture parity in adw-m4-03 via a payload
rewrite: `cwd → runDir` (so cLens's `.clens` walk-up resolves to the run
dir) and `transcript_path →` a host copy pulled out of the container by
`liveContainerTranscriptFetch` (a `docker cp` edge). The clens seam is
SYNCHRONOUS:

    container?: { fetchTranscript: (containerPath: string) => string | undefined }
    // src/observability/clens.ts

For the remote kind the transcript lives IN the sandbox; the only way to pull
it out is `sandbox.files.read(path)`, which is **async**. The synchronous
seam cannot host it, so adw-m5-02 shipped remote capture as an explicit
deferral: both the main remote lane (`runTicket`) and — after adw-m5-03 — the
remote CI round route their capture runDir correctly but wire NO
`fetchTranscript`, so the in-sandbox session's `cwd`/`transcript_path` stay
in-sandbox paths and capture downgrades to `ok:false`.

## Scope

Refined 2026-09-11, operator-approved. **Decision: the kind-neutral async
seam, two bindings.** The rejected alternative — widening
`ClensHooksConfig.container.fetchTranscript` in place to
`=> string | undefined | Promise<...>` — is the smaller diff, but it leaves
the config key named `container` while it hosts E2B, and every kind then pays
await-per-final-event on the clens hot path for a rewrite only two kinds need.

## Requirements

**The seam** (`src/observability/clens.ts`)

- [x] Replace `ClensHooksConfig.container` with a kind-neutral
      `transcriptFetch?: { fetch: (path: string) => Promise<string | undefined> }`.
      One seam, two bindings — not a wrapper per kind (Art. VIII).
- [x] `makeClensHooks` awaits the fetch inside its already-async forward
      callback. The rewrite semantics are otherwise byte-identical to the
      container path proven in adw-m4-03: on a host copy, forward with
      `cwd → runDir` and `transcript_path →` the copy; on `undefined`, forward
      the original path and record the
      `transcript could not be copied out` error (reworded — it is no longer
      container-specific).
- [x] `liveContainerTranscriptFetch` keeps its docker-cp behavior and returns
      a Promise. Its host-copy destination (`<runDir>/.clens/transcripts/`)
      and its 0700 / owner-only invariants are unchanged.

**The E2B binding**

- [x] New `liveE2bTranscriptFetch(sandbox, runDir)` backed by
      `sandbox.files.read(path)`, writing the transcript to the SAME
      `<runDir>/.clens/transcripts/` destination as the container path.
- [x] A read failure resolves `undefined` — it never throws into the hook
      callback. A wedged fetch must not strand a run.

**Wiring — BOTH remote lanes**

- [x] `runTicket`'s main remote lane binds the E2B fetch (`src/cli.ts`).
- [x] The remote CI round binds it too (`src/pipeline/nodes/ci-round.ts`) —
      the mini-lane deferred in adw-m5-03. Missing either lane leaves half the
      metric red.

**Invariants that must survive (each already cost a live finding)**

- [x] `.clens` stays 0700 and transcripts owner-only; no token appears in a
      fetched artifact.
- [x] The LAST (Stop/SessionEnd) fetch decides capture health (m4-06).
- [x] An EARLIER event's fetch failure stays tolerated — do not reopen the
      breaker at SessionStart (m4s-005).

## Verify

- [x] Contract-level, fake async edge: a remote session whose final-event
      transcript copy succeeds drives `checkCaptureCompleteness` → `ok:true`.
- [x] The container path is **unregressed** — the adw-m4-03 suite stays green
      against the renamed seam.
- [x] `bun run lint && bunx tsc --noEmit && bun test` (see run log: the 2–4 pre-existing real-subprocess timeouts fail identically on main).
- [ ] **Live bar, operator-gated and BILLED:** a remote-run scratch ticket
      journals `capture ok:true` with the transcript copied out and no token
      in the artifact. E2B is selected declaratively (`just run-remote`); it is
      never the default. If the offline work lands green and only this bar is
      owed, carve it out rather than leaving the ticket `in-progress` — the
      adw-m5-03 precedent.

## Out of scope

The remote CI round itself (adw-m5-03); parallel runs; `adw-m7-04` (the Codex
resume-path stamp — Tier 1.2, cheap once this shape is settled).

## Run log

**2026-09-11 — run `adw-m5-06-remote-capture-parity-1789078287643` (worktree,
claude) was KILLED mid-`build`.** No `run-end`, `abort`, `watchdog` or
`hard-stop` event: the process died hard, so the CLI's finalize-blocked path
never ran and the ticket was left `in-progress` with `attempts: []`. Reset to
`queued` by hand — this is the one case the "no terminal path leaves a ticket
in-progress" guarantee cannot cover.

`plan` completed (65 236 tokens, 43 turns); both cLens captures `ok:true`. The
`build` agent had written 407 lines across the right six files before it died.
That work was verified by hand — lint clean on `src test`, typecheck clean,
815/818 tests with the three failures identical to main's baseline — and then
DISCARDED, so Tier 1.1 lands as one unbroken factory run rather than a salvage.

The run also surfaced `adw-selfhost-lint-gate` (fixed, commit `84a7371`): the
lint gate was vacuously red for every self-target worktree run, so this ticket
could not have reached a green `gates` node regardless of the code.

**2026-09-11 — run `adw-m5-06-remote-capture-parity-1789085359632` reached
`gates` and BLOCKED at 89.4 min** (plan 12.9m → build 30.7m → test 41.9m →
gates 3.8m). The factory finalized correctly this time: `status: blocked` with
a proper `attempts:` entry. That entry is LEFT AS IS — it records a blocked
factory attempt, which is what happened, and `src/intake/attempts.ts` owns the
block.

The block was a factory defect, not a defect in this ticket's code. `bun test`
writes test NAMES to stderr; two of this repo's names contain the literal
`command not found` (they are the tests for the config-error heuristic), and
`isConfigError` (`gates.ts:87`) regexes gate stderr for exactly that. So any
non-zero suite exit is misread as "the command could not run" → terminal
`fail`, **zero repair rounds charged**. Filed as `adw-gates-config-error-regex`.
It only fires because the suite is not green on this host — the second,
independent blocker.

Because the lane never reached its commit/push/open-pr nodes, **PR #1 was
opened by hand** from the run's worktree after verifying lint (83 files), tsc,
and 132/132 across clens/e2b/ci-round. Status set to `in-review` to reflect
reality. NOTE: `sync-pr-state` will skip this ticket rather than reconcile it,
because the PR is not recorded in `attempts:` — that is deliberate, not an
oversight. Close it out by hand when the PR merges.

PR: https://github.com/silouone/adw-factory/pull/1

**2026-09-11 — MERGED.** PR #1 squash-merged as `924251a` (+526/-99, 7 files).
Re-verified on merged main: lint 83 files, tsc clean, 132/132 across
clens/e2b/ci-round. Worktree and branch removed. Status set to `done` BY HAND —
`sync-pr-state` skips this ticket because the PR is not in `attempts:` (the
entry records the blocked factory attempt, which is what actually happened).

**Still owed — carved out, not silently dropped:** the operator-gated, BILLED
live bar. A remote run must journal `capture ok:true` with the transcript
copied out and no token in the artifact. Offline parity is proven; live parity
is not.

This is a DISTINCT bar from the one adw-m5-03 owes. m5-03 owes a resumed
repair round in a reattached sandbox; this ticket owes `capture ok:true` with
the transcript actually copied out. Both are billed and operator-gated, and
neither is tracked yet: `adw-m5-07-remote-ci-round-live-bar` is created by
`adw-tier0-ledger-reconcile`, which has not run. Until it does, this
paragraph IS the record — the adw-m5-03 precedent: code-complete is not
live-verified, and the ledger must say so rather than imply otherwise.
