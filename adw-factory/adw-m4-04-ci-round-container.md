---
id: adw-m4-04-ci-round-container
type: feat
status: done
priority: 3
created: 2026-07-17
epic: adw-m4
depends: [adw-m4-03-container-auth]
attempts: []
---
# CI round for container-run tickets

> Filed from the M4 milestone two-axis review (spec finding a1): S4.3 says
> the workspace lifecycle "shall be identical across kinds from the
> pipeline's point of view", but the CI mini-lane is worktree-only. The gap
> was documented honestly in adw-m4-02/03 yet had no tracking ticket — this
> is it. Not part of the M4 epic exit (the chore lane ends at open-pr);
> queued for pickup before container isolation becomes the default.
>
> **Refined 2026-07-18** (operator-approved decisions recorded below).
> Updated for what landed after filing: adw-m4-07 replaced `pushEnv?()` with
> `Workspace.push()` (the "amendment #2 reaches the CI round" note in the
> original Context is obsolete — the reused push node now delegates to the
> workspace kind), and adw-m4-06 added `hardStop?()`, exec `timeoutMs`, and
> the reclaim breadcrumb. No plan §5 amendment is needed: the Workspace
> contract is untouched; everything here is kind-internal seams plus
> `CiRoundDeps` growing edges the CLI already holds.

## Context

`src/pipeline/nodes/ci-round.ts` rehydrates a kept workspace for the CI
maintenance hook via `attachWorktree` + a host-side
`existsSync(attempt.workspace)` — both meaningless for a container attempt
(workspace `/work/repo`, existence = the kept container). A container-run
ticket whose PR goes red in CI currently lands "state error, no round
charged".

Most machinery already exists: `attachContainer` returns a workspace whose
`exec` (with the m4-06 kill ceiling), `agentSpawn` (in-container repair
agent via `baseOptions`), and `hardStop` all work — the mini-lane engine
inherits Art. V containment and S4.5 spawn-in-container by construction.
What's missing: kind detection + a container existence/state probe, an
auth-carrying reattach whose `push` is real (today's attach handle is
deliberately inert), and capture parity (the round routes capture to
`dirname(attempt.workspace)` = the nonsense host path `/work`, and the CLI's
CI `makeHooks` binding lacks the m4-03 `container.fetchTranscript` config).

## Operator decisions (2026-07-18)

1. **Reattach the kept container** — never re-provision from the PR branch:
   the recorded SDK session (`query({resume})`, S2.4 "same session") lives
   inside the kept container; a fresh container would break resume.
2. **An EXITED kept container is `docker start`ed** before the round (host
   reboot / docker restart is routine over an in-review PR's lifetime; the
   checkout and session file survive a stop). A failed start → state error,
   no round charged. A MISSING container → state error, no round charged
   (same shape as the missing-worktree path).
3. **Missing `CLAUDE_CODE_OAUTH_TOKEN` → skip, no charge, stay in-review**:
   a worktree-launched run (no container preflight) can hit a container
   attempt's CI round; the sync line reports the reason naming the env var
   and how to relaunch. `main()` passes `containerAuth` whenever the token
   is present in the launch env — no longer gated on the current run's
   `--isolation` flag (the container *preflight refusal* stays
   container-dispatch-only).

## Requirements

- [x] **Kind detection + probe:** ci-round recognizes a container attempt
      (`attempt.workspace === CONTAINER_WORKDIR`, the same marker clean
      uses) and probes existence/state via the container
      (`containerName(attempt.runId)`), not host `existsSync`. Missing
      container → blocked, **no round charged**, reason names the container.
      Worktree attempts stay byte-identical.
- [x] **Exited → start:** an existing-but-not-running container is
      `docker start`ed before rehydration; a failed start → blocked, no
      round charged, reason carries the docker stderr.
- [x] **Auth-carrying reattach:** the round's workspace handle carries the
      launch-env `claudeToken` (agentEnv → the resumed in-container agent),
      a **freshly minted** git token (the per-round `gh auth token` thunk),
      and an origin **re-captured from the HOST repo and re-validated**
      (`validateOriginUrl`) at reattach time — never read from the
      agent-writable checkout (the m4-07 trust rule). Its `push` is the
      existing `hostSidePush` (no new push code); `agentSpawn` + `hardStop`
      + exec `timeoutMs` ride as they already do on `makeContainer`.
- [x] **Missing agent token:** container attempt + no `claudeToken` →
      sync outcome `skipped`, no charge, ticket stays in-review, reason
      names `CLAUDE_CODE_OAUTH_TOKEN` and the relaunch fix.
- [x] **Capture/tracing parity:** for a container attempt the round's
      capture runDir is the prior run's dir `join(runsRoot, attempt.runId)`
      (not `dirname(attempt.workspace)`), for BOTH `makeHooks` and
      `checkCaptureCompleteness`; the CLI's CI `makeHooks` binding gets the
      same `container.fetchTranscript` (docker-cp against the kept
      container, host copies into that runDir) the main lane got in m4-03.
      Worktree attempts keep `dirname(attempt.workspace)` byte-identical.
- [x] **CLI wiring:** `runTicket`'s CI binding passes the container edges +
      auth through `CiRoundDeps` (every edge a fake in tests, Art. I);
      `main()` supplies `containerAuth` whenever the token is present.
- [x] **Charge discipline unchanged:** all state checks (probe, start,
      token, session-id) happen BEFORE the charge; the round is still
      charged first once rehydration is possible (S2.4 crash-safety).
- [x] **Live verify:** a container-run scratch ticket with a deliberately
      CI-red PR completes one in-container repair round — repair agent runs
      IN the kept container (resumed session), re-push lands via the host
      edge on a fresh token, capture ok:true on the round's journal, no
      token in the container.

## Build protocol

Red-first at these seams (validator-gated per the build protocol):

1. **container.ts:** `containerState(name)` probe (running/exited/missing),
   `startContainer(name)`, and the reattach variant binding auth + a
   host-repo-captured validated origin (rejected origin throws
   `WorkspaceError` — loud, Art. IX). Docker edges follow the existing
   docker-gated suite pattern; pure argv/config helpers unit-tested.
2. **ci-round.ts:** container branch of the rehydration flow behind
   injected `CiRoundDeps` container edges (fakes in tests): probe/start/
   token-skip/no-charge orderings, capture runDir routing, charge-first
   pin, worktree path unchanged.
3. **cli.ts:** CI `makeHooks` container binding (fetchTranscript against
   `containerName(attempt.runId)`), `containerAuth` no longer gated on the
   current run's isolation flag.
4. Full suite + lint + tsc green; two-axis /code-review vs the pre-ticket
   base; then the live bar on the scratch repo.

## Out of scope

Remote/E2B (adw-m5); parallel runs; the m4-07 ssh-push residue (its own
small ticket).

## Result (2026-07-18)

Done. 511 tests, lint + tsc green. Red set validator-APPROVED before source;
two-axis /code-review (base ddc232d): no hard standards violations, spec
faithful — 2 findings applied red-first/under-green (see below). Commits:
2eb0db0 refinement → 7e1d641 red → 6dcc48b green → e169dc7 review fixes →
8b1d853 live finding.

**Live bar (scratch, m4s-010, PR #13):** forcing ticket (ci-fail.marker;
local gates green, remote CI red) built under `--isolation container`; the
CI round then ran from a WORKTREE-default launch (flag-independence proven
live). Round: charge-first commit (`ciRounds:1`) → repair agent IN the kept
container on the RESUMED session (its reply quotes the original ticket from
its own context — the resume proof) → agent judged the failure intentional
and changed nothing → E8 `--allow-empty` retrigger → host-edge re-push on a
per-round minted token (retrigger commit at the remote branch tip) →
capture ok:true on the round's journal with the transcript docker-cp'd out
(the validator's named blind spot, the CI `fetchTranscript` binding, proven
live). Token-at-rest scans clean: PID-1 env, container fs, no credential
helper, no origin remote in the checkout. Second CI failure then blocked
the ticket live ("single repair round is spent", S2.4) — the cap holds for
container attempts. Container reclaimed via `adw clean --ticket m4s-010`;
PR #13 left OPEN for operator disposal.

**Review fixes:** (1) daemon-outage ≠ missing — `containerState` now routes
through pure `interpretInspect` (the interpretGh precedent): only a genuine
"no such container/object" is a `missing` verdict; any other inspect
failure throws loudly, so a docker-daemon outage can never finalizeBlock a
healthy in-review ticket. (2) origin-capture dedup: `captureValidatedOrigin`
shared by provision + reattach (Art. VIII).

**Live findings:**
- PRE-EXISTING clean bug, bites worktrees too (fixed red-first, 8b1d853):
  finalizeBlocked appends a terminal attempts entry with the SAME
  runId/workspace as the original, and clean's ref collection had no dedup —
  the second teardown of the one kept workspace threw "already reclaimed?"
  AFTER the reclaim (non-zero exit, stack trace instead of the report).
- Bun `spawnSync` sets `result.error` even for plain non-zero exits (unlike
  Node) — only a null `status` means the subprocess never ran. The
  containerState edge tests status, not error.
- A resumed repair agent treats a DELIBERATELY-forced CI failure as
  intentional (it remembers the forcing ticket) and takes the E8 retrigger
  path — a marker-forcing koan therefore always ends blocked-after-cap, by
  design of the resume semantics. Budget for that in future forcing runs.
