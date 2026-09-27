---
id: adw-m4-06-hard-stop-container
type: feat
status: done
priority: 3
created: 2026-07-17
epic: adw-m4
depends: [adw-m4-05-review-fixes]
attempts: []
---
# Container hard-stop, reclaim, and capture-downgrade hardening

> Refined 2026-07-18 (operator-approved scope decisions) from the M4 Codex
> findings deferred past adw-m4-05, TRIMMED when adw-m4-07 landed (High-4 rw
> mount + origin-URL validation moved there, done). Cites `S<n>` = adw-v1.md §3
> · `N<n>` = §4 · `E<n>` = §5 · `Art.<n>` = constitution. This is a
> spec-complete refinement; the two amendments below need landing in the plan +
> contract before implementing (the amendment rule).

## Problem Statement

Three runtime-containment gaps remain in the container isolation kind after the
credential boundary was closed (adw-m4-05/07):

1. **The Art. V hard stop never reaches the in-container agent (High-3).** The
   run's wall-clock deadline aborts an internal `AbortController` in the engine,
   but that signal is only observed at node boundaries and, mid-stream, between
   SDK message yields (`consumeAgentStream`). It is never handed to the Agent
   SDK (`Options.abortController`), and the container spawn hook
   (`spawnClaudeCodeProcess`) drops the `signal` the SDK forwards to it. A
   cooperative claude stops on stdin-EOF, but a **runaway or prompt-injected
   agent wedged in a loop — not blocked on stdin — keeps running after the
   deadline**, because killing the host-side `docker exec` client does not cross
   into the container (proven in the m4-02 spike). There is no real OS-level
   stop.

2. **A crash/deadline-stranded container cannot be reclaimed (High-3, reclaim).**
   `adw clean` reclaims only containers recorded in a ticket's `attempts[]`, and
   that entry is written **after** the lane returns (`finalizeTerminal`). A run
   that dies before finalize (SIGKILL, a crash, or a hard stop that leaves the
   process wedged) orphans its container: `clean` has nothing to find.

3. **`Workspace.exec` has no timeout.** A gate or git command that hangs inside
   the container (a wedged test process, a network stall) blocks the lane
   forever — the deadline timer fires but the awaited `exec` never returns, so
   the boundary check that would honor it is never reached.

Separately, a capture-integrity gap (Medium-2): when the container transcript
`docker cp` fails on a Stop/SessionEnd hook event, the clens forwarder passes
the **unreadable in-container `transcript_path`** to cLens unchanged and leaves
capture health `ok:true` — a silent capture hole (`src/observability/clens.ts`).

## Solution

A container run gets a **graceful-then-hard stop** and a **reclaim breadcrumb**:

- **Hard stop = abortController + `docker kill` backstop (operator-approved).**
  The engine's abort/deadline signal is threaded to the SDK's
  `Options.abortController`, and the spawn hook forwards its `signal` to the
  `docker exec` child. On breach: the SDK tears the query down (closing stdin →
  a *cooperative* claude exits and flushes its transcript), and after a brief
  grace the factory issues `docker kill` (SIGKILL PID-1) so a *wedged/runaway*
  agent is really stopped. The container is left **exited, not removed** — still
  `inspect`/`cp`/`commit`-able, so Art. VII ("kept for autopsy, reclaimed only
  by explicit clean") holds; the real `docker rm -f` stays `adw clean`'s job.
  This is the one containment that delivers a genuine Art. V hard stop without
  destroying autopsy evidence.

- **Reclaim breadcrumb = a run-dir marker (operator-approved).** At provision
  the factory writes a small `runs/<runId>/workspace.json`
  (`{kind, containerId, branch}`); `adw clean` additionally scans run dirs and
  reclaims a container that exists but has no reclaiming `attempts[]` entry.
  This amends clean.ts's "never blindly scans `runs/`" rule — deliberately (see
  Amendments).

- **`Workspace.exec` timeout.** `ExecOptions` gains an optional `timeoutMs`; a
  command that exceeds it is killed and returns a non-zero `ExecResult` naming
  the timeout, so a hung in-container gate cannot wedge the lane. A generous,
  configurable default (not a fixed short value) — long-but-legitimate gates (a
  full test suite) must not trip it.

- **Capture downgrade (Medium-2).** When `config.container` is set and a
  Stop/SessionEnd event carries a `transcript_path` that `fetchTranscript`
  cannot copy out (returns `undefined`), the forwarder appends a `capture`
  `ok:false` downgrade for that session (last-record-wins, breaker opened) and
  does NOT forward the unreadable in-container path. Ordered so it runs against
  the killed-but-present container after a hard stop (Art. VI).

From the operator's point of view: a timed-out container run now actually stops,
its container is always reclaimable by `adw clean`, and a capture that could not
be read is flagged rather than silently trusted.

## User Stories

1. As the operator, I want a container run that breaches its wall-clock deadline
   to actually stop the in-container agent process, so that a runaway or
   prompt-injected agent cannot keep consuming resources or acting after the
   ceiling (Art. V).
2. As the operator, I want the stop to try graceful shutdown first (SDK
   teardown → stdin-EOF), so that a cooperative agent flushes its transcript
   before it is killed (Art. VI).
3. As the operator, I want a `docker kill` backstop after a brief grace, so that
   an agent that ignores stdin-EOF is still hard-stopped.
4. As the operator, I want the hard-stopped container left EXITED (not removed),
   so that it stays available for autopsy and `adw clean` remains its only
   reclaimer (Art. VII).
5. As the operator, I want the SDK to receive the run's abort signal, so that
   the ceiling interrupts the agent stream promptly rather than only at the next
   message yield.
6. As the operator, I want `adw clean` to reclaim a container whose run crashed
   or was killed BEFORE its `attempts[]` entry was written, so that no orphaned
   container is ever unreclaimable.
7. As the operator, I want the reclaim breadcrumb written at provision time, so
   that even a provision-time or build-time crash leaves a reclaimable trail.
8. As the operator, I want a hung in-container gate/git command to time out
   rather than wedge the lane forever, so that the deadline is actually
   enforceable end-to-end.
9. As the operator, I want the exec timeout default to be generous and
   configurable, so that a legitimately long gate (a full test suite) is not
   killed mid-run.
10. As the operator, I want the worktree kind's behavior unchanged (no timeout
    tripping, no docker-kill path, no breadcrumb it doesn't need), so that this
    change is scoped to the container boundary and does not regress M1.
11. As the operator, I want a container transcript that cannot be copied out of
    the container to downgrade capture health to `ok:false`, so that an
    unreadable capture is flagged, not silently forwarded as an in-container
    path with `ok:true` (Art. VI, Medium-2).
12. As the operator, I want the capture downgrade to use the established
    last-record-wins + open-the-breaker semantics (adw-m3-06/m6-04), so that it
    composes with the existing capture-health machinery.
13. As a security reviewer, I want a test proving a wedged in-container process
    (one that ignores stdin-EOF) is terminated by the `docker kill` backstop on
    a deadline breach, so that the hard-stop core is regression-guarded — not
    re-filed as a documented limitation.
14. As a maintainer, I want the pipeline node graph unchanged and the abort
    plumbing to ride the existing engine→node→query seam, so that isolation
    kinds stay indistinguishable from the pipeline's point of view (S4.3).

## Implementation Decisions

- **Abort threading rides the existing `AgentQuery` seam.** `AgentQueryOptions`
  (`build.ts`) gains an optional `signal?: AbortSignal`; the agent nodes thread
  `ctx.signal` into it. *(Amended at implementation, validator-flagged:
  `ctx.signal` is ALWAYS present, so BOTH kinds' options carry it — the kinds
  stay indistinguishable through the seam (Story 14) and the ceiling reaches
  the SDK for worktree runs too (Story 5); the original "worktree options stay
  byte-identical" parenthetical was wrong.)* `live-query.ts` maps it onto the
  SDK's `Options.abortController` (wrapping the signal) AND forwards the spawn
  hook's `signal` to the `docker exec` child `spawn(...)`. The pure
  `containerSpawnArgv` is untouched; only the impure spawn wiring changes.
  Data-only seam keeps every kind testable without the real SDK (Art. IX).

- **The `docker kill` backstop lives behind the Workspace, not the engine.** A
  new optional `Workspace.hardStop?(): Promise<void>` (container-only) issues
  `docker kill` on the run's container; the engine invokes it after a brief
  grace on ANY blocked exit taken with the run signal aborted, when the
  workspace exposes it (worktree has none → graceful abort only, unchanged).
  *(Refined at the live bar: "the engine's abort path" was literally the
  boundary-breach `abort()` — but a mid-stream deadline surfaces as the agent
  node's graceful `fail`, which reaches blocked without a boundary check; live
  m4s-005's container survived its breach until the backstop moved to the
  blocked exit, guarded on `signal.aborted` so a normal blocked run keeps its
  workspace untouched for autopsy.)* This keeps the engine kind-blind (S4.3) —
  precedent: `agentSpawn`, `pushEnv`→`push`. **plan §5 Workspace-contract
  amendment.**

- **`Workspace.exec` timeout is a contract change.** `ExecOptions` gains
  `timeoutMs?: number` with a generous module default; both kinds honor it at
  their subprocess edge (`Bun.spawn` + a kill timer). **plan §5 amendment** (the
  `exec` signature/contract). Worktree and container share the same default.

- **The reclaim breadcrumb is a run-dir file.** Written at provision
  (`runs/<runId>/workspace.json` = `{kind, containerId, branch, ticketId}` —
  `ticketId` added at implementation, validator-endorsed: without it clean
  cannot honor `--ticket` for a crashed run nor apply the S4.4 active-status
  safety skip, and the skip is load-bearing — a LIVE run always has a
  breadcrumb, so a concurrent unnamed `adw clean` would otherwise kill a
  healthy running container); `adw clean` gains a run-dir scan that reclaims a
  live container with no reclaiming `attempts[]` entry, under the SAME
  ticket-scoping rules as attempts refs. **clean.ts invariant amendment**
  ("clean never blindly scans `runs/`" → "clean reclaims from `attempts[]` AND
  from provision-written run-dir breadcrumbs"). Keep the breadcrumb minimal and
  provision-written so a build-time crash is still covered (Story 7).

- **Capture downgrade is a narrow edit to the clens forwarder.** In
  `forward`, when `config.container` is set, the event is **Stop or
  SessionEnd** (matching the Solution's wording), and `input.transcript_path`
  is a string but `fetchTranscript` returns `undefined`: append the `capture`
  `ok:false` downgrade (reusing `sessions`/breaker machinery), mark the session
  failed, and return `{continue:true}` WITHOUT forwarding the in-container
  path. *(Amended at the live bar: this decision originally said any event —
  live m4s-005 showed early events routinely fire before the in-container
  transcript exists, so an any-event downgrade opened the breaker at
  SessionStart and killed the whole session's capture. Each fetch overwrites
  the same host copy, so only the FINAL events' fetch decides whether the
  transcript is readable; an earlier failure keeps the pre-m4-06 tolerance.)*

- **Deferred (explicitly out of THIS ticket):** full ssh push support (the
  m4-07 ssh-residue) — ssh already fails gracefully; guaranteeing a host key
  reaches the push edge is a separate credential-availability concern, deferred
  to a follow-up (operator-approved). The CI-round container reattach stays
  adw-m4-04.

## Testing Decisions

- **What a good test is here:** it drives the stop/reclaim/timeout/downgrade
  through the existing seams and observes external behavior — is the process
  gone, is the container reclaimable, did the hung command return, was capture
  flagged — never internal call shapes. Real docker + real git are the I/O
  boundary (docker-gated suites skip loudly, run for real in verify).

- **Hard-stop, docker-gated (`container.contract.test.ts` /
  `cli.container.test.ts`).** The load-bearing security test (Story 13): start a
  container, run a process that IGNORES stdin (`sleep infinity` / a tight loop),
  trigger the abort/deadline, assert `hardStop()` (`docker kill`) leaves the
  container EXITED and the in-container process gone — and the container still
  `inspect`-able (kept for autopsy). Prior art: the m4-02 kill-semantics spike,
  the m4-07 security tests.

- **Abort threading unit seam (`live-query.test.ts`, no docker).** Assert the
  SDK options carry an `abortController` derived from the signal, and the
  container spawn wiring forwards the hook `signal` to the child — pure/faked
  SDK, mirroring the existing `containerSpawnArgv`/`agentSdkEnv` unit tests.

- **Reclaim breadcrumb (`cli.container.test.ts` + `clean` tests).** A container
  whose `attempts[]` was never written (simulate a crash: provision, remove the
  ticket attempt, leave the container) is still reclaimed by `runClean` via the
  run-dir breadcrumb; a normal recorded attempt is not double-reclaimed
  (idempotent).

- **`Workspace.exec` timeout (both kinds, contract-level).** A command that
  sleeps past `timeoutMs` returns non-zero naming the timeout and is actually
  killed (no lingering process); a fast command with the same option is
  unaffected; the default lets a multi-second command through.

- **Capture downgrade (`clens.test.ts`, no docker).** With `config.container`
  set and a `fetchTranscript` stub returning `undefined` for a Stop event: a
  `capture` `ok:false` downgrade is journaled for that session (last-record-
  wins), the in-container path is NOT forwarded, and the breaker opens. Prior
  art: the m3-06/m6-04 capture-health tests.

## Out of Scope

- Full ssh push support (deferred follow-up; m4-07 ssh-residue).
- CI-round container repair reattach (adw-m4-04).
- Remote/E2B isolation and its stop/reclaim (adw-m5).
- The deeper dropped-event-without-count capture case (m6-04 cLens-side
  limitation, unchanged).

## Live re-verification bar

After green (precedent m4-03/05/07): a live container run on the scratch repo
whose deadline is forced to breach mid-agent must (a) leave the container EXITED
with the in-container agent process gone (`docker kill` fired), (b) be reclaimed
by `adw clean` even if the attempt was never recorded, and (c) if a transcript
cp is forced to fail, journal an `ok:false` capture downgrade.

## Result (2026-07-18) — done, live re-verified

485 tests, lint+tsc green. Two-axis /code-review: no hard standards
violations; both axes' silent-kill finding fixed red-first (journal
`hard-stop` event + operator line). All three live-bar criteria proven on
the scratch repo:

- **(a)** run `m4s-005-1784377105692` (caps 1 min, agent mid-flight at
  breach): `⛔ hard stop — container killed`, container EXITED exit=137
  (SIGKILL), `hard-stop ok:true` journaled, kept inspect-able, then
  reclaimed by `adw clean --ticket`.
- **(b)** run `m4s-006-1784377213484` SIGKILLed mid-agent (ticket stuck
  in-progress, `attempts: []`, container running): reclaimed via the
  provision-written breadcrumb with `--ticket`; the unnamed sweep spared it
  while in-progress and reported nothing after (idempotent).
- **(c)** run `m4s-009-1784377744750` (green path, in-container transcript
  deleted in a loop): Stop-time cp failed → `capture ok:false` downgrade
  journaled after the healthy record (last-record-wins), the in-container
  path never forwarded. PRs #10–12 opened by the forcing runs remain for
  operator disposal.

**The live bar caught two real gaps the unit red set structurally missed**
(both fixed red-first, see amendments above): the hard stop originally rode
only the boundary `abort()` while a mid-stream deadline surfaces as build's
graceful fail (first m4s-005 run's container survived its breach, still
running); and the any-event capture downgrade opened the breaker at
SessionStart (transcript not yet on disk) and killed every live session's
capture. A third live fact worth recording: **Stop/SessionEnd hooks do NOT
fire on the abort-teardown path**, so a hard-stopped run's transcript is
never docker-cp'd — it survives only inside the kept exited container
(autopsy via manual `docker cp`), and the Stop-scoped downgrade governs
normally-ending sessions.
