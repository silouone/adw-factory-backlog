---
id: adw-m5-04-remote-run-robustness
type: feat
status: done
priority: 3
created: 2026-07-19
epic: adw-m5
depends: [adw-m5-02-e2b-workspace]
attempts: []
---
# Remote run robustness: un-wedge teardown, liveness watchdog, sandbox sizing

> Spec minted from the first real-target remote live bar (run
> `clens-006-distill-io-seam-1784414282079`, 2026-07-19) — the m4-07
> precedent: a live finding converted to a full SDD spec, not silently
> patched. Seams operator-approved 2026-07-19 (adapter timeout + build-node
> watchdog + target-config sizing; NO new engine seam).

## Problem Statement

The operator dispatched a real clens ticket with `--isolation remote` and
the run stranded: the in-sandbox agent went permanently silent 52 seconds
into the build (after 25 flawless research turns — transcript recovered
from the paused sandbox), and the factory could not stop it. The 45-minute
ticket cap fired on schedule but had no effect — the run only ended because
E2B's 1-hour lifetime ceiling paused the sandbox, and even then the host
lane process hung a further 15+ minutes until the operator killed it by
hand. A bounded, self-terminating factory (Art. V) must never depend on the
operator or on the cloud provider's backstop to end a run. Separately, the
default sandbox (2 CPU / 1024 MB) is suspected too small for a real target
(clens + `bun install` + the agent binary), which is the leading suspect
for the silent stall itself — the tiny scratch target ran green in 41 s on
the same spec.

## Solution

Three defenses, each at an existing seam, so a wedged remote agent is
detected within minutes, terminated through the existing hard-stop path,
and honestly journaled as a blocked outcome — and so a real target gets a
sandbox sized for its workload. The deadline chain becomes effective
end-to-end: cap fires → agent query aborted → teardown completes even over
a dead transport (synthesized exit after a bounded grace) → build returns
FAIL → the existing single hard-stop site runs → lane finalizes blocked →
sandbox reclaimable by `adw clean`. A stream-liveness watchdog catches the
wedge long before the deadline would.

## User Stories

1. As the operator, I want a remote run whose agent stream dies to end as a
   blocked run within minutes, so that I never find a lane hanging hours
   later.
2. As the operator, I want the ticket's `caps.minutes` deadline to actually
   terminate a remote run, so that the cap I write in a ticket is a real
   bound and not advisory.
3. As the operator, I want the lane process itself to always exit after a
   terminal outcome, so that a dispatch is safe to launch unattended
   (nohup, cron, CI).
4. As the operator, I want a wedged agent detected by output silence (no
   SDK stream message for N minutes), so that a stall is caught long before
   the wall-clock deadline burns the whole cap.
5. As the operator, I want the watchdog trip recorded in the journal with a
   descriptive reason (ticketId, node, silence duration), so that a
   post-mortem needs no transcript archaeology.
6. As the operator, I want the same watchdog to protect container and
   worktree runs, so that a hung local agent is bounded by the same rule
   and the kinds stay behaviorally symmetric.
7. As the operator, I want the SDK teardown to complete even when the E2B
   transport is dead (kill acknowledged by nothing), so that the abort
   signal chain cannot itself hang the lane.
8. As the operator, I want the hard stop to remain single-sited at the
   blocked exit, so that the Art. V kill path stays auditable in one place.
9. As the operator, I want a watchdog-tripped run to still attempt the
   workspace hard stop, so that a possibly-alive agent process is killed
   and cannot keep consuming the sandbox.
10. As the operator, I want a per-target sandbox size in the target config,
    so that a heavy repo (clens) gets the memory it needs while the scratch
    target stays cheap.
11. As the operator, I want a raised factory default for sandbox memory,
    so that an unconfigured real target does not silently run in a
    1 GB sandbox.
12. As the operator, I want the requested size verified against what E2B
    actually provisioned (getInfo) at the live bar, so that a silently
    clamped tier limit is a recorded fact, not an assumption.
13. As the operator, I want the stranded-run failure mode covered by unit
    tests against the existing fakes, so that a regression in the teardown
    chain is caught without a live sandbox.
14. As the operator, I want the blocked finalization to append the attempt
    record even when build ended by watchdog/deadline, so that `adw clean`
    can reclaim the workspace through the normal attempts path (the
    breadcrumb remains the backstop).
15. As the operator, I want re-dispatching the reset clens ticket to be the
    live bar for this spec, so that the fix is proven on the exact workload
    that exposed the gap.
16. As the factory's future remote CI round (adw-m5-03), I want teardown
    and watchdog semantics settled now, so that the resumed-session repair
    round inherits a bounded build primitive.
17. As the operator, I want the operator stream to print a distinct line
    when a run ends by watchdog or deadline (with the ⛔ hard-stop line
    precedent), so that a live-watched dispatch tells me what bounded it.
18. As a ticket author, I want an optional per-ticket stall override
    alongside the existing caps, so that a legitimately slow ticket (long
    gates, big installs) can widen the silence tolerance without touching
    factory defaults.

## Implementation Decisions

- **No new seams.** All three defenses ride existing injected edges: the
  e2b background-command fake (spawn adapter tests), the build node's
  agent-query seam (fake stream tests), the `E2bOps.create` edge + target
  loader schema (provision/config tests). The engine is untouched; plan §5
  Workspace contract is untouched (no amendment needed — `E2bCreateOpts`
  is kind-internal).
- **Adapter teardown grace (remote kind):** when the SDK teardown invokes
  kill on the adapted process — or closes stdin and the underlying command
  has not settled — a bounded grace timer is armed; if the real exit has
  not arrived when it lapses, the adapter synthesizes the exit event
  (null code, idempotent with any late real settlement) so the SDK query
  loop always unblocks. The stdin write/close callbacks are likewise
  bounded so a dead transport cannot hang the teardown before kill is even
  reached. Grace is a named constant (tens of seconds), not configurable.
- **Build-node liveness watchdog (all kinds):** a timer reset on every SDK
  stream message; on trip, the build node aborts the agent query through
  the same abort-controller channel the deadline uses and resolves FAIL
  with a descriptive error carrying ticketId, node name, and the silence
  duration (Art. IX error discipline). Default silence tolerance is a
  named constant (minutes); the ticket contract's `caps` gains an optional
  stall-minutes field as the override channel, defaulted like the existing
  caps fields.
- **Hard stop stays single-sited:** the blocked-exit hard-stop guard is
  widened from "run signal aborted" to also cover a watchdog-tripped build,
  so a wedge and a deadline breach take the same kill path. No second
  hard-stop call site.
- **Journal + operator surface:** a watchdog trip and a synthesized-exit
  teardown are journaled as events (hard-stop event precedent) and echoed
  as operator lines, so the bounding cause is in both records.
- **Sandbox sizing:** the target config gains an optional remote sandbox
  size (cpu count, memory MB), ignored by non-remote kinds; the provision
  factory threads it through `E2bOps.create`. The factory default memory is
  raised to a 4 GB-class value; the implementation verifies the requested
  size against the E2B tier's actual allowance during the live bar (a
  clamped grant is recorded, not assumed).
- **Blocked finalization ordering:** the attempt record for a
  watchdog/deadline-bounded run is appended through the existing terminal
  finalizer so reclaim-by-attempts works; the provision-time breadcrumb
  remains the crash backstop (m4-06 decision, unchanged).

## Testing Decisions

- Good tests here assert **external behavior**: the build outcome, the
  journal events, the exit event's arrival and timing, the create call's
  received size — never adapter internals or timer bookkeeping.
- **Spawn adapter:** against the existing background-command fake — a
  never-settling wait with a kill call must produce exactly one exit event
  after the grace (fake timers); a late real exit after synthesis is
  ignored; bounded stdin close on a dead transport.
- **Build node:** against the existing fake agent-query stream — a stream
  that stalls mid-run trips the watchdog, aborts through the query's
  abort-controller, and resolves FAIL with the descriptive reason; a
  healthy slow-but-talking stream does NOT trip; the blocked exit invokes
  hard stop for a tripped run (m4-06 suite precedent).
- **Config/provision:** loader accepts/defaults/rejects the sandbox-size
  field (existing target-loader validation suite shape); provision passes
  the size to the create edge (existing fake `E2bOps` assertions).
- **Ticket contract:** the optional stall-minutes caps field parses,
  defaults, and rejects garbage (existing caps parser tests as prior art).
- **Live bar:** re-dispatch the reset clens ticket `--isolation remote`
  with the sized sandbox — the discriminating probe for the 52-second
  stall's root cause. Success = green PR or an honestly blocked run that
  self-terminates within the watchdog bound with the lane process exiting
  on its own. Verify provisioned size via getInfo; verify sandbox
  reclaim/pause state afterwards.

## Out of Scope

- Root-cause certainty for the 52-second silent stall — the sized rerun IS
  the discriminating experiment; if it stalls again at 4 GB, that becomes
  its own investigation (E2B stream vs API hang).
- E2B capture parity (the sync `fetchTranscript` seam / async sandbox read
  mismatch) — already an explicit m5-02 deferral, separate follow-on.
- The remote CI round (adw-m5-03) and SSH push residue (adw-m4-08).
- cLens-side capture durability (the durable-write-ack wishlist item).
- Any engine-seam deadline hook (explicitly declined at seam review — the
  adapter + watchdog pair is the chosen containment).

## Amendment (2026-07-19, implementation-discovered)

**Sandbox sizing is a TEMPLATE property in the E2B API, not a create-time
option.** Verified against e2b@2.35.0 (the latest release, npm-checked
2026-07-19): `cpuCount`/`memoryMB` exist only on the template-build request
(`BasicBuildOptions`, defaults 2 CPU / 1024 MB — exactly the grant the
stranded run recorded), and neither `SandboxOpts` nor the server's
`NewSandbox` request schema carries any sizing field. The spec's "provision
factory threads it through `E2bOps.create`" is therefore realized as: the
target's `sandbox: { cpus, memoryMB }` selects a **pre-built sized template
ref** (pure `e2bTemplateRef`: default size → the pinned `adw-agent:<ver>`;
custom → `adw-agent-<c>c<m>m:<ver>`), passed through the SAME injected
`E2bOps.create` edge (its `templateRef` argument), where the unit tests
observe it. The raised factory default (story 11) is realized as the build
script's `memoryMB` default (`E2B_TEMPLATE_MEMORY_MB = 4096`; CPU stays 2) —
the live bar rebuilds the pinned template at that size and verifies the grant
via `getInfo` (story 12). Operational consequence: a custom-sized target
needs its sized template built once
(`bun scripts/e2b-template-build.ts --cpus N --memory-mb M`) before its first
remote dispatch; an unbuilt ref fails provision loudly. Seams, stories, and
tests are otherwise unchanged; no plan §5 amendment needed (`E2bCreateOpts`
stays kind-internal, the Workspace contract untouched).

## Resolution (2026-07-19) — DONE, superseded by adw-m5-05

The two findings below were carried to [[adw-m5-05-settlement-bound]] and RESOLVED:
finding B → the post-abort settlement bound (bound the query as a whole at the
node); finding A → **root-caused as E2B's 60-second default per-command timeout**
killing the un-`timeoutMs`'d agent command (NOT the memory/API/egress hypotheses
this ticket chased) — fixed by `timeoutMs: E2B_SANDBOX_TIMEOUT_MS` on the agent
spawn. Proven GREEN live on the exact stranding workload (clens-006, PR
silouone/clens#17). This ticket's SHIPPED defenses (teardown grace, liveness
watchdog, engine hard-stop widening, template sizing) remain in place — necessary
and now sufficient in composition with m5-05.

## Live-bar result (2026-07-19) — BLOCKED: bar failed, two findings

Run `clens-006-distill-io-seam-1784454102565`, dispatched 09:41:42Z on the
rebuilt 4096 MB template (grant verified via getInfo: 2 CPU / 4096 MB,
unclamped — story 12 ✅). Outcome: **the run wedged again and did NOT
self-terminate** — the spec's own success bar ("an honestly blocked run that
self-terminates within the watchdog bound with the lane process exiting on
its own") was not met. Lane killed by operator-proxy SIGTERM at cap+4 min,
sandbox reclaimed via `adw clean --ticket` (breadcrumb path, again proven),
clens ticket reset to queued (clens 1a1a0e7).

1. **The 52-second stall is NOT memory.** The in-sandbox transcript froze at
   09:42:50Z — 52 s into the session, the identical signature as the 1024 MB
   strand, now at 4× memory. The agent process itself ended at 09:45:05Z
   (E2B envd log). Salvaged evidence:
   `runs/clens-006-distill-io-seam-1784454102565/salvaged-in-sandbox-transcript.jsonl`
   (77 lines; last entries: healthy research turns → tool_result → silence
   mid-stride). The discriminating experiment discriminated: sizing is
   ruled out; prime suspects now E2B command-stream death vs an agent-side
   API hang at a deterministic point.
2. **The SDK teardown wedge sits ABOVE the seams this ticket bounded.**
   Both the liveness watchdog (due ~09:53Z) and the 45-min deadline
   (10:26:42Z) aborted the derived controller, yet the journal shows no
   synthesized-exit, no watchdog fail, no abort — the SDK never reached the
   spawn adapter's stdin-close/kill (no grace ever armed), so the bounded
   chain never started. Leading hypothesis: the streaming-input teardown
   first sends an interrupt control_request over stdin and awaits its
   control_RESPONSE from the (dead) agent — an unbounded await upstream of
   every seam we own. The unit suite structurally cannot see this (fake
   streams reject on abort; the real SDK's iterator does not). Fix
   direction for the follow-on: bound the query as a WHOLE at the node
   (race the stream against a post-abort settlement timeout — no SDK
   cooperation required), and/or verify the SDK's abort semantics against
   its source; the declined engine-seam hook may deserve re-review with
   this evidence.

What the live bar DID prove: template sizing end-to-end (config → sized ref
→ create → getInfo), the 4096 MB default grant, breadcrumb reclaim of a
wedged run's sandbox, and a valid S2.6 journal cut under SIGTERM. The unit
layer (613 tests) covers the shipped defenses; they are necessary but now
proven insufficient for this wedge.

## Further Notes

- Evidence base: journal + salvaged in-sandbox transcript of run
  `clens-006-distill-io-seam-1784414282079`; E2B getInfo records (pause at
  the 1 h ceiling to the second; 2 CPU/1024 MB grant); the stranded lane
  killed by operator SIGTERM with a valid S2.6 journal cut; reclaim via
  `adw clean --ticket` (Phase 4 wiring proven live in the same incident).
- The container kind is immune to the strand (killing the local docker-exec
  client unblocks the SDK) but gains the watchdog for symmetric bounding.
- Constitution: Art. I (red tests first, validator-gated), Art. V (bounded
  execution — this spec closes the remote gap), Art. IX (pure helpers,
  injected edges, descriptive errors). Charge discipline unchanged.
- clens-006-distill-io-seam sits queued in the clens repo as the designated
  live-bar workload for this spec.
- Validator observation (red-gate round, recorded not hidden): the CI
  mini-lane's agent node (`ci-round.ts`) still passes `ctx.signal` raw — it
  is NOT watchdog-bounded. Remote CI rounds are adw-m5-03 scope (currently
  reported as an honest skip), so the gap is unreachable for the remote kind
  today; fold the watchdog into the mini-lane when m5-03 lands.
