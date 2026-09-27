---
id: adw-m5-03-remote-ci-round
type: feat
status: done
priority: 3
created: 2026-07-18
epic: adw-m5
depends: [adw-m5-02-e2b-workspace]
attempts: []
---
# CI round for remote-run tickets

> Coarse ticket — refine requirements at pickup (plan §8). Minted during
> the adw-m5-02 refinement (operator decision 2026-07-18: clean wiring in
> m5-02, CI round deferred here) — the m4-04 precedent: the S4.3 gap is
> tracked, not hidden, and adw-m5-02 reports it honestly at the CLI.
>
> **Refined 2026-07-19** (operator-approved decisions below). The remote
> analog of adw-m4-04, reusing every seam that landed after this ticket was
> filed: `provisionE2b`/`makeE2b` (adw-m5-02), the trusted host-side push
> edge `hostSidePush` (adw-m4-07 parity), the finding-A `timeoutMs` on the
> e2b agent-spawn adapter, and the liveness watchdog + settlement bound
> (adw-m5-04/05). No plan §5 amendment: the `Workspace` contract is
> untouched; everything here is kind-internal seams plus `CiRoundDeps`
> growing one edge the CLI already holds the auth for.

## Scope

The remote analog of adw-m4-04: a remote-run ticket whose PR goes red in
CI gets its repair round in the SAME sandbox on the RESUMED session.
Reattach = `Sandbox.connect(sandboxId)` (resuming the paused-on-keep,
filesystem-only sandbox); kind detection via the `E2B_WORKDIR` marker;
existence probe = a non-resuming `Sandbox.list` membership check (paused
counts); capture runDir routing; re-push via the shared host-side edge on a
per-round minted token; missing `E2B_API_KEY`/`CLAUDE_CODE_OAUTH_TOKEN` →
skip, no charge, in-review kept (the m4-04 shape). Charge discipline
unchanged (checks before charge, charge-first once rehydration is
possible). Cites S2.4, S4.3, S4.5.

## Operator decisions (2026-07-19)

1. **Reattach the kept sandbox — never re-provision from the PR branch.**
   The recorded SDK session the round resumes (`query({resume})`, S2.4 "same
   session") lives inside the kept sandbox; a fresh sandbox would break
   resume. Reattach = `ops.connect(sandboxId)`, which RESUMES the
   filesystem-only paused sandbox (m5-02 keep=pause was chosen partly to make
   this possible). The `sandboxId` is server-assigned, read from the run's
   reclaim breadcrumb (`runs/<runId>/workspace.json`, adw-m5-02) — unlike a
   container name, it is not derivable from the runId, so reattach is async.

2. **Binary state model — no `stopped→start` analog.** E2B sandbox state is
   `running | paused` (SDK `SandboxState`); a paused sandbox auto-resumes on
   `connect`, so there is no `docker start` step. The existence probe is a
   **non-resuming** `Sandbox.list({state:['running','paused']})` membership
   check by `sandboxId` — NOT `Sandbox.getInfo`, which RESUMES a paused
   sandbox (SDK doc: "If the sandbox is paused, it will be resumed"). Present
   (running or paused) → run the round; **missing → blocked, no round
   charged** (same shape as the missing-worktree / missing-container path).

3. **Two skip conditions (E2B needs both keys).** The remote round needs
   BOTH `CLAUDE_CODE_OAUTH_TOKEN` (→ the resumed in-sandbox agent's only
   auth) AND `E2B_API_KEY` (the SDK cannot probe/connect the sandbox without
   it). **Either missing → sync `skipped`, no charge, ticket stays
   in-review**, reason naming the missing var + the relaunch fix. (Container
   had only the one token gate; the auth is already isolation-independent —
   a worktree-default launch that meets a remote attempt's round serves or
   cleanly skips it, `deps.containerAuth` reused as the agent-auth channel.)

4. **Capture parity DEFERRED (amends the Scope line "fetchTranscript").**
   The round routes its capture runDir to `join(runsRoot, attempt.runId)`
   for BOTH `makeHooks` and `checkCaptureCompleteness` (the in-sandbox
   workspace path has no meaningful host `dirname`, same as the container
   kind), but does NOT wire an E2B `fetchTranscript`: E2B's `files.read` is
   ASYNC and the clens `container.fetchTranscript` seam is SYNCHRONOUS. The
   ci-repair session therefore journals `capture ok:false` exactly like the
   main remote lane already does (adw-m5-02 deferral). Wiring parity only
   here would make the two remote lanes asymmetric — the sync→async seam is
   its own cross-cutting ticket **adw-m5-06** (fixes both lanes at once).

5. **Watchdog + settlement bound fold-in.** The ci-repair node today calls
   `consumeAgentStream(stream, ctx)` with NO watchdog (ci-round.ts) — a
   remote ci-repair agent that wedged would strand the mini-lane. It now
   builds a `makeStreamWatchdog("ci-repair", ctx)`, threads `watchdog.signal`
   into the query options, and passes the watchdog to `consumeAgentStream`
   (the build/repair-node shape). This benefits ALL kinds (a strict
   improvement). Finding-A's `timeoutMs` needs NO ci-round change: the
   resumed agent spawns via `workspace.agentSpawn` → the e2b adapter →
   `defaultE2bBackground`, which already carries `timeoutMs` — so a correct
   `reattachE2b` handle (agentSpawn descriptor present) inherits the fix for
   free. A + B compose exactly as in the main lane.

## Requirements

- [x] **Kind detection + probe:** ci-round recognizes a remote attempt
      (`attempt.workspace === E2B_WORKDIR`) and probes existence via the
      injected non-resuming edge (paused counts). Missing sandbox → blocked,
      **no round charged**, reason names the sandbox. Worktree/container
      attempts stay byte-identical.
- [x] **Missing keys → skip:** remote attempt with either
      `CLAUDE_CODE_OAUTH_TOKEN` or `E2B_API_KEY` absent from the launch env →
      sync outcome `skipped`, no charge, ticket stays in-review, reason names
      the missing var and the relaunch fix. Checked BEFORE any SDK probe.
- [x] **Auth-carrying reattach (`reattachE2b`):** post-charge, the round's
      workspace handle is a FULL `makeE2b`-shape workspace — `agentSpawn`
      descriptor (`{kind:"e2b", sandboxId, workdir}`, the resume crux), the
      launch-env `claudeToken` (→ `agentEnv`), a **freshly minted** per-round
      git token (→ the host-side push only), and an origin **re-captured from
      the HOST repo and re-validated** (`validateOriginUrl`) at reattach time
      (the m4-07 trust rule) — never read from the agent-writable checkout.
      Its `push` is the existing `hostSidePush` (no new push code). A missing
      breadcrumb / rejected origin throws `WorkspaceError` naming the ticket
      (loud, Art. IX).
- [x] **Capture runDir parity:** for a remote attempt the round's capture
      runDir is `join(runsRoot, attempt.runId)` for BOTH `makeHooks` and
      `checkCaptureCompleteness`. No E2B `fetchTranscript` wired (deferred to
      adw-m5-06) — capture degrades `ok:false` like the main remote lane.
      Worktree/container runDir routing byte-identical.
- [x] **Watchdog fold-in:** `ciRepairNode` builds a `makeStreamWatchdog(
      "ci-repair", ctx)`, passes `watchdog.signal` into the query options and
      the watchdog into `consumeAgentStream` (adw-m5-04/05 settlement bound).
- [x] **CLI wiring:** `runTicket`'s CI binding passes the remote edges +
      auth through `CiRoundDeps` (every edge a fake in tests, Art. I) over
      `realE2bOps()`; `hasAuth` additionally requires `E2B_API_KEY` in the
      launch env.
- [x] **Charge discipline unchanged:** all state checks (keys, existence,
      session-id) happen BEFORE the charge; the round is still charged first
      once rehydration is possible (S2.4 crash-safety).
- [x] **Live verify:** CARVED OUT 2026-09-11 to
      `adw-m5-07-remote-ci-round-live-bar` — see "Carve-out" below. The bar
      itself is NOT met.
      Original: a remote-run scratch ticket with a deliberately
      CI-red PR completes one in-sandbox repair round on the RESUMED session
      — repair agent runs IN the reattached sandbox, re-push lands via the
      host edge on a fresh token, no token in the sandbox at rest.

## Status note (2026-07-19)

Code-complete and offline-green (634 tests, tsc+lint). **Live bar (box 128)
still OWED** — stays `in-progress`. The 16:11 remote run that closed adw-m5-02
(`clens-006-distill-io-seam-1784470261685`) went green **first try**: gates
passed, NO CI-red PR, NO repair round — so it does NOT exercise this ticket's
resumed-session repair path. Reconfirmed the two things to budget for on the
billed live run: the **m4-04 koan** (a resumed agent may read a marker-forced CI
failure as intentional → blocked-after-cap by design) and the **paused-sandbox
survival window** (Hobby's ~1 h fs-only pause ceiling may make `missing →
blocked (no charge)` the COMMON path, not the edge — a retention-tier / eager-
round design call, not a bug). Operator holds the keys for this billed run.

## Build protocol

Red-first at these seams (validator-gated per the build protocol):

1. **e2b.ts:** `E2bOps` grows `connect(sandboxId): Promise<E2bSandbox>` and
   a non-resuming `sandboxExists(sandboxId): Promise<boolean>` (via
   `Sandbox.list`; the "connect unneeded" note updated); `reattachE2b(runId,
   ticketId, {hostRepo, runsRoot, auth, ops})` reads the breadcrumb
   sandboxId, connects/resumes, re-captures+validates the HOST origin, and
   returns the FULL `makeE2b` handle (agentSpawn + auth + origin +
   hostSidePush) — NOT the inert `makeAttached`. Pure argv/config helpers
   unit-tested; the SDK edge follows the injected-fake pattern (no unit test
   touches the network).
2. **ci-round.ts:** remote branch of the rehydration flow behind an injected
   `CiE2bOps` on `CiRoundDeps` (fakes in tests): key-skip / existence /
   no-charge orderings, capture runDir routing, charge-first pin,
   worktree+container paths unchanged; the watchdog fold-in on `ciRepairNode`.
3. **cli.ts:** CI `CiRoundDeps` remote binding (`ciE2bOps` over
   `realE2bOps()`); `hasAuth` requires both keys.
4. Full suite + lint + tsc green; two-axis /code-review vs the pre-ticket
   base; then the operator-gated (billed) live bar on the scratch repo.

## Out of scope

E2B capture parity (the sync→async `fetchTranscript` seam — adw-m5-06);
parallel runs; the m4-08 ssh-push residue.

## Verify

Contract-level: a fake-edge ci-round suite mirroring the m4-04 set; live:
a remote-run scratch ticket with a CI-red PR completes one in-sandbox
repair round on the resumed session.

## Result (2026-07-19) — OFFLINE COMPLETE, live bar operator-gated

Contract level DONE, `bun run lint && bunx tsc --noEmit && bun test` green
(632 tests). Commits: `55be6d8` refinement → `2fb4924` red → `f6b29b8` green
→ (review fix) on `main`, NOT pushed.

Seams: `e2b.ts` — `E2bOps` grew `connect` (resumes the fs-only paused sandbox)
+ `sandboxExists` (non-resuming `Sandbox.list` membership, paused counts, never
`getInfo`); `reattachE2b` returns the FULL `makeE2b` handle (agentSpawn resume
crux, claudeToken→agentEnv, per-round gitToken→push, host-recaptured+validated
origin); shared `resolveE2bCrumb` dedups the breadcrumb read (attach/reattach,
Art. VIII). `ci-round.ts` — remote branch mirroring the container branch,
charge-first ordering, capture runDir routing (NO fetchTranscript — parity
deferred to adw-m5-06), watchdog + settlement-bound fold-in on `ciRepairNode`
(all kinds benefit); the whole mini-lane's engine timers ride the one injected
seam. `cli.ts` — `ciE2bOps` over `realE2bOps()`, isolation-independent agent
auth, `E2B_API_KEY` required.

Red-gate validator-APPROVED before source; caught two real red-set defects
(a permanently-red push assertion — `fakeE2bWorkspace` hardcoded `adw/x`; a
MISSING host-origin re-validation refusal test) — both fixed before green.
Two-axis `/code-review` (base `b68c1df`): no hard standards violations, spec
faithful; one PARTIAL applied red-first — the skip reason now names EXACTLY the
absent key(s) (`CiE2bOps.hasAuth` bool → `missingAuth: string[]`) per decision 3.

**Finding-A `timeoutMs` carries to the resumed ci-repair agent for free** — it
spawns via `workspace.agentSpawn` → the e2b adapter → `defaultE2bBackground`,
which already carries `timeoutMs`; `reattachE2b`'s handle keeps the descriptor,
so no ci-round change was needed for A (B, the watchdog/settlement bound, is the
added containment backstop).

**`sandboxExists` pagination VERIFIED against the compiled SDK** (the one piece
of real logic no unit test exercises — the fakes bypass the paginator): in
`node_modules/e2b/dist/index.js`, `Paginator._hasNext = true` at construction
(the `while (hasNext)` loop runs at least once), and `Sandbox.list()` with no
args returns "running and paused by default" (doc L4603-4613) — so a PAUSED kept
sandbox appears in the membership check. Non-resuming (never `getInfo`). Assert
these by probe if the SDK is ever bumped.

**Still owed — the operator-gated (billed) live bar:** a remote-run scratch
ticket with a deliberately CI-red PR completes one in-sandbox repair round on
the RESUMED session (recipe: `env -u GITHUB_TOKEN -u GH_TOKEN bun src/cli.ts run
--target <t> --isolation remote --ticket <id>`, then force a CI-red PR and let
the next run's sync hook drive the round). Ticket stays `in-progress` until it
passes. Two things to budget for:
- **The m4-04 koan** (finding c): a resumed agent recognizes a marker-forced CI
  failure as intentional and takes the E8 retrigger path → blocked-after-cap by
  design.
- **The paused sandbox must still be reattachable at CI-red time.** The kept
  sandbox pauses fs-only under the 1 h Hobby lifetime ceiling; if a realistic
  PR-open → CI-red → re-run delay exceeds it, `missing → blocked (no charge)` is
  the COMMON path, not the edge — a design conversation (retention tier / eager
  round), not a bug. Verify the survival window on the live bar.

## Carve-out (2026-09-11)

Tier 0.4 of `ai_docs/2026-09-10-deep-state-and-roadmap.md`: *"no ticket
lingers in-progress."* This ticket sat `in-progress` from July (`fe601fd`)
as a marker meaning "code complete, live bar owed" — but `in-progress` means
"a run is believed in flight", and the CLI refuses to dispatch such a ticket
(`activeRunRefusal`). The status was blocking the ticket it described.

Offline work is complete and green. The owed **live** bar is inherently
remote — reattaching a paused E2B sandbox and resuming the recorded session —
so there is no local form of it; a worktree run exercises a different branch
of `ci-round.ts` entirely. It is therefore operator-gated and BILLED, and has
moved to `adw-m5-07-remote-ci-round-live-bar` rather than holding this ticket
open forever.

**Code-complete is not live-verified, and this ticket does not claim to be.**
