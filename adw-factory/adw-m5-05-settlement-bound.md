---
id: adw-m5-05-settlement-bound
type: feat
status: done
priority: 3
created: 2026-07-19
epic: adw-m5
depends: [adw-m5-04-remote-run-robustness]
attempts: []
---
# Post-abort settlement bound: end a wedged remote stream with zero SDK cooperation

> Minted from the adw-m5-04 live bar (run
> `clens-006-distill-io-seam-1784454102565`, 2026-07-19), which FAILED its own
> self-termination bar and left two findings (recorded in that ticket's
> "## Live-bar result"). This ticket carries **finding B** (the containment
> hole) to a fix and **finding A** (the 52-s stall) to a named root cause.
> adw-m5-04 is thereby SUPERSEDED, not reopened (m4-07 precedent: a live
> finding → a fresh SDD spec).

## Problem Statement

The adw-m5-04 defenses (adapter teardown grace, liveness watchdog, engine
hard-stop widening, sandbox sizing) are shipped and green (613 tests) but were
proven **insufficient** on the live bar: a wedged remote agent stream did not
self-terminate. Both the watchdog trip (~09:53Z) and the 45-min deadline
(10:26:42Z) aborted the build node's derived controller, yet the journal shows
no synthesized-exit, no watchdog fail, no abort — the SDK's own teardown parked
in an unbounded internal await **above** every seam adw-m5-04 bounded, so the
adapter's stdin-close/kill grace never even armed. Confirmed against the pinned
SDK (`@anthropic-ai/claude-agent-sdk@0.3.209`): the streaming-input teardown's
`interrupt()` issues a `control_request` and awaits its `control_response`
(`Query.request(...).response`) with no timeout; against a dead agent that
response never arrives, and `hasBidirectionalNeeds()` (true whenever hooks are
set) forces the path. The unit suite structurally cannot see this — the fakes
REJECT on abort, the real SDK iterator does NOT.

Separately, finding A: the 52-s in-sandbox stall reproduced identically at
4 GB (sizing ruled out). Its root cause is unknown — prime suspects are E2B
command-stream death vs an agent-side API hang at a deterministic point — and
the e2b adapter drops the agent's own stderr (only `onStdout` is wired), so its
dying words are lost.

## Solution

**Bound the query as a whole at the build node**, needing zero SDK cooperation.
After the node's derived controller aborts (watchdog trip OR `ctx.signal`), race
the stream's settlement against a named-constant timeout
(`POST_ABORT_SETTLEMENT_MS`, tens of seconds, reusing the injected `TimerSeam`).
If the iterator has not thrown or ended when it lapses: **ABANDON** it — stop
awaiting, do NOT call `iterator.return()` / do NOT await SDK cleanup — and return
the watchdog trip FAIL (or the run-abort FAIL) so the engine's single-sited hard
stop → blocked finalize → `adw clean` path runs and the lane exits. This lives
INSIDE the build node (shared by repair via `consumeAgentStream`); the engine is
untouched (the engine-seam deadline hook was declined at seam review and this
fix does not need it). Manual iteration replaces `for await` precisely because
`for await` would call `iterator.return()` on exit and re-park in the wedged
teardown.

Fold in: **e2b adapter stderr capture** — wire `onStderr` through
`E2bBackgroundImpl` / `makeE2bSpawnHook` to an injected sink (default
`process.stderr`, the container `inherit` precedent), so a wedge's diagnostics
survive.

Then, at the live bar, root-cause finding A with a directed one-sandbox probe
(no full lane): run claude directly with `onStdout` AND `onStderr`, replay the
first 52 s, and watch WHERE it hangs (network egress vs agent-internal). Convert
whatever it names into a finding — a spec/ticket if factory-side, or a recorded
"E2B upstream" disposition + mitigation if platform-side.

## User Stories

1. As the operator, I want a remote agent whose stream wedges (the SDK teardown
   never settling) to end as a blocked run within the watchdog + settlement
   bound, so that no lane strands on an unbounded SDK await.
2. As the operator, I want the settlement bound to need zero SDK cooperation, so
   that a future SDK teardown regression cannot re-open the strand.
3. As the operator, I want a stream that DOES settle promptly after abort (the
   cooperative shape) to keep taking the existing abort/trip path unchanged, so
   that the bound adds latency only to a genuinely dead transport.
4. As the operator, I want the abandoned-iterator path to leave no lingering
   timer that itself keeps the lane alive, so that the process still exits.
5. As the operator, I want the in-sandbox agent's stderr surfaced (operator
   stream and/or run dir), so that a wedge's dying words are available for
   post-mortem instead of dropped.
6. As the operator, I want the 52-s stall carried to a NAMED root cause with
   recorded evidence (or a recorded, operator-accepted "E2B upstream"
   disposition + mitigation), so that finding A is closed, not carried forever.
7. As the operator, I want a live re-dispatch to prove the lane process EXITS on
   its own (shell-return, no SIGTERM) — node-return-FAIL is necessary but not
   the bar.

## Implementation Decisions

- **No new engine seam, no engine behavior change.** The bound rides the build
  node's existing watchdog-derived abort channel; the engine's deadline /
  hard-stop / blocked logic is unchanged. (`engine.ts` gains ONLY the shared
  `realUnrefTimer` constant relocated there beside `TimerSeam` — the Art. VIII
  dedup, since the settlement bound is a second unref'ing call site.) If the live bar
  proves a node-level bound genuinely cannot exit the process (an abandoned SDK
  transport holding a ref'd handle), an explicit terminal `process.exit` is an
  **operator decision** (the handoff's note), proposed before the live bar — not
  a silent divergence.
- **Settlement on the watchdog.** `makeStreamWatchdog` gains an `abandoned`
  promise (armed via `config.setTimer` when the derived signal aborts; resolves
  after `POST_ABORT_SETTLEMENT_MS`) and its `cancel()` clears BOTH the liveness
  arm and the settlement arm. The settlement timer is `unref`'d as
  defense-in-depth. The CI mini-lane (no watchdog) keeps no settlement bound —
  it is watchdog-unprotected until adw-m5-03 (recorded).
- **Manual iteration.** `consumeAgentStream` drives `stream[Symbol.asyncIterator]()`
  and races each `.next()` against `watchdog.abandoned`; on the abandon marker it
  returns `watchdog.failure()` (the trip) or the run-abort FAIL, WITHOUT calling
  `.return()`. The per-message decisions (init/turn-ceiling/result/abort) are
  unchanged, but converting from `for await` drops its implicit
  `iterator.return()` on early exit — so the cooperative early-exit paths
  (mid-flight abort, turn ceiling, is_error) now REQUEST the SDK teardown
  fire-and-forget (never awaited, so a wedged teardown cannot hang the node) to
  avoid leaking a still-live agent. The abandon path alone requests NO teardown
  (the teardown is known wedged; even requesting it can hold a ref'd handle).
  Both facts are pinned by the `returnCalled()` spy.
- **stderr sink.** An injected `onStderr` on `E2bSpawnHookOpts` (default
  `process.stderr.write`), threaded to `commands.run`'s `onStderr` via a widened
  `E2bBackgroundImpl` — symmetric with `onSynthesizedExit`.

## Testing Decisions

- Good tests assert **external behavior**: the build outcome and its marker,
  never timer bookkeeping. Fake timers drive the settlement clock.
- **Two named fakes make the crux legible:** the existing `stallingQuery`
  REJECTS on abort (→ the existing catch path, unchanged) vs a new `wedgedQuery`
  that IGNORES the abort and NEVER settles (→ the settlement path). The gap
  between them IS this ticket.
- **Wedge red:** `wedgedQuery` yields healthy messages then never settles; fire
  the watchdog timer (derived signal aborts) then the settlement timer → the
  node returns the watchdog FAIL (`watchdog: true`, ticketId + node named). The
  "red" against current code is a HANG (the `for await` parks → the test's own
  timeout) — the faithful reproduction; brief the validator.
- **Cooperative-abort:** after a prompt post-abort settle, assert the settlement
  arm was CANCELED (no timer outlives the stream) — the existing "healthy stream
  cancels all arms" test never aborts, so it cannot catch a leaked settlement
  timer.
- **Both abandon branches:** a watchdog-trip wedge → watchdog FAIL; a
  `ctx.signal` (deadline) wedge → abort FAIL (not misclassified as a trip).
- **Repair** inherits the bound (shared `consumeAgentStream`).
- **stderr:** against the widened background fake, `onStderr` data reaches the
  injected sink.
- **Live bar:** re-dispatch a real remote workload (the reset clens-006, or a
  deliberately wedge-y probe). Success = a green PR OR an honestly blocked run
  that self-terminates within the bound **with the lane process exiting on its
  own** (shell-return timed, no SIGTERM). Batched with the finding-A probe into
  one operator-gated live round.

## Live bar — GREEN (2026-07-19) — DoD met, DONE

Re-dispatched the EXACT workload that stranded twice last night
(`clens-006-distill-io-seam`, `--isolation remote`, run
`clens-006-distill-io-seam-1784470261685`). Outcome: **GREEN** — a clean full
lane (dispatch → provision → assemble-prompt → build → gates → commit → push →
open-pr → run-end green), the host process EXITED ON ITS OWN (exit 0, no
SIGTERM). The build node ran the full distill chore PAST the ~52–60 s point
where both prior runs froze; all clens gates passed (lint + typecheck + 2182
tests, 0 failures); PR opened at https://github.com/silouone/clens/pull/17
(ticket → in-review). The journal carries ZERO watchdog / synthesized-exit /
hard-stop events — no strand, no wedge, no abort: **fix A removed the cause,
so fix B (the backstop) never had to engage** (exactly as predicted). DoD #1
(a live re-dispatch self-terminates), #2 (finding A named + fixed + live-proven),
and the settlement bound (unit-proven) are all satisfied. adw-m5-04 SUPERSEDED.

## Finding A — ROOT-CAUSED + FIXED (2026-07-19, live-proven)

**The "52-s stall" was E2B's 60-second default per-command timeout killing the
agent, NOT an API/egress hang.** A directed one-sandbox probe (run claude
directly + a parallel egress heartbeat to `api.anthropic.com`) proved it:

- Run 1 (agent `commands.run` with NO `timeoutMs`, as the factory shipped): the
  agent made healthy 200 API turns and egress stayed fast (~110 ms, code 401)
  the WHOLE time — then the agent process was killed at **+60 176 ms** with
  `TimeoutError: [deadline_exceeded] … exceeding 'timeoutMs' — the total time a
  long running request (like command execution) can be [limited]`.
- Run 2 (same probe, agent `commands.run` WITH a generous `timeoutMs`): the
  agent ran healthily PAST 60 s — 200 turns at +106/+122/+124/+132/+133 s,
  egress healthy throughout — until MY 150 s observation window closed. No kill,
  no stall.

Confirmed in the pinned SDK (`e2b@2.35.0`): `Commands.start` applies
`timeoutMs ?? defaultProcessConnectionTimeout` where
`defaultProcessConnectionTimeout = 60_000`. The factory's `defaultE2bBackground`
(the agent-spawn adapter) passed no `timeoutMs`, so every remote agent session
exceeding ~60 s was killed mid-flight. The `commands.run` exit event and the
SDK's `query()` iterator are independent: the process dying mid-stream sends the
SDK into its OWN teardown, which — SUSPECTED, from the 0.3.209 source read, not
live-observed — parks on `interrupt()` awaiting a `control_response` from the
dead agent, wedging the `for await` regardless of the adapter's synthesized
exit. That makes A the TRIGGER and B (the settlement bound) the containment for
the CONSEQUENCE. The 40 s scratch toy (m5s-001) fit under 60 s; clens did not.
`execIn` (gate commands) already overrode this default; the agent path was
simply missed.

**Fix:** the agent `commands.run` now carries `timeoutMs: E2B_SANDBOX_TIMEOUT_MS`
(the sandbox-lifetime ceiling — the factory's OWN bounds, engine deadline +
watchdog + settlement, govern the agent's lifetime, never E2B's per-command
default; that a 1 h command cap is honored across a full 45-min run is ASSUMED,
bounded first by the engine deadline and the 1 h sandbox pause — the probe
proved health only to ~150 s). Lifted ABOVE the `E2bBackgroundImpl` seam as a
REQUIRED field so the cap is un-forgettable and unit-observable (the
keepMemory:false lesson). Fixes A and B compose: A removes the cause of the
strand; B bounds any OTHER future wedge.

## Live-round staging (offline prep, 2026-07-19 — makes the operator-gated round one-shot)

**Fix B is sufficient for the lane to EXIT (DoD #1), by construction.** The CLI
entry is `process.exit(await main())` (`cli.ts:972`). `process.exit()` is a
FORCIBLE termination — it ignores dangling timers/sockets — so the abandoned e2b
transport cannot keep the process alive. The old 75+ min strand was NOT a
dangling-handle leak: the build node's `for await` parked on the wedged
`.next()`, so the lane never resolved, `await main()` never returned, and
`process.exit` was never reached. The settlement bound makes the build node
RETURN a FAIL → lane resolves → `blocked()` runs `hardStop` (a fresh
`Sandbox.connect` + `commands.kill` of the in-sandbox claude PID) and resolves →
`main()` resolves → the forcible exit fires. No `process.exit`-after-finalize
fallback patch is needed; the live bar CONFIRMS rather than discovers. (If, and
only if, the live bar shows the process not returning to the shell, the fallback
is a `process.exit` after the terminal finalize — an operator decision, staged.)

**Finding A (the 52-s stall) — sharpened hypothesis + one-shot probe.** Only run
2's salvaged transcript survived
(`runs/clens-006-distill-io-seam-1784454102565/salvaged-in-sandbox-transcript.jsonl`,
77 lines). Analysis: healthy SUB-SECOND turns (35 assistant / 24 user,
~23 Bash+Read tool round-trips) until an abrupt freeze at 09:42:50.541Z
(~53 s in) IMMEDIATELY after a SUCCESSFUL small `tool_result` (2.3 KB, no
error) — the model's NEXT turn simply never arrived. Accumulated context at the
freeze ≈ 250 KB transcript (≈ 60–70 K tokens — moderate, not a context limit).
So the stall is a HUNG API TURN, not an agent loop or a tool failure; the
deterministic ~52–53 s twice points at the Nth request once accumulated context
crosses a threshold, or a burst-rate effect after ~23 rapid round-trips. Prime
suspect: E2B egress to `api.anthropic.com` stalling a request; secondary:
agent-side hang. One-shot directed probe (minutes, one sandbox, no lane):
provision like the factory (`provisionE2b` or `Sandbox.create("adw-agent:2.1.210")`
→ relock → clone clens → `bun install`), run claude DIRECTLY via `commands.run`
(background, stdin, `onStdout` AND the now-wired `onStderr`; `ANTHROPIC_LOG=debug`
/ `--debug`), replay the research reads to accumulate ~60 K context, and watch
for the ~52 s freeze. When it hangs, from a SECOND `commands.run`:
`curl -v https://api.anthropic.com/...` during the hang + a large streaming POST
— egress alive with the agent's request stuck ⇒ egress/API hang; claude silent
with no request in flight ⇒ agent-internal. Convert the result to a finding
(spec/ticket if factory-side; recorded "E2B upstream" disposition + mitigation
if platform-side). Auth: `CLAUDE_CODE_OAUTH_TOKEN` from the token FILE
`~/.adw/claude-token` (never print it), `E2B_API_KEY` from repo `.env`.

## Out of Scope

- Any engine-seam deadline hook (declined at adw-m5-04 seam review; re-propose
  as an amendment only if the live bar proves the node-level bound cannot exit).
- The remote CI mini-lane's watchdog/settlement (adw-m5-03).
- E2B capture parity (`capture ok:false` on remote — m5-02 deferral).
- Root-cause CERTAINTY for finding A if it proves an E2B platform bug — the
  deliverable is then recorded evidence + upstream report + a factory-side
  mitigation decision.
