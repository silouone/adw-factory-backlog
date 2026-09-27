---
id: adw-m5-02-e2b-workspace
type: feat
status: done
priority: 3
created: 2026-07-14
epic: adw-m5
depends: [adw-m5-01-e2b-template]
attempts: []
---
# E2B Workspace implementation

> Refined at pickup 2026-07-18 (plan §8, protocol §3) — operator-approved
> decisions recorded below. Contingent on the adw-m5-01 spike proving the
> `SpawnedProcess`-over-E2B adapter; if the spike fails, this ticket goes
> back to refinement.

## Context (grounded in source, 2026-07-18)

- `AgentSpawnSpec` is container-only (`src/workspace/types.ts:57`) and
  `live-query.ts` maps it onto the SDK's `spawnClaudeCodeProcess` via
  docker-exec argv — the remote agent must run IN the sandbox (S4.5), so
  this ticket carries **plan §5 amendment #6** (below).
- Kind detection keys on the workspace-path marker: `clean.ts`
  (`keptWorkspace`, attempts probe) and `ci-round.ts:302` compare against
  `CONTAINER_WORKDIR`; `parseWorkspaceBreadcrumb` accepts only
  `kind:"container"`. The remote kind needs its own marker + breadcrumb
  kind, and clean wiring.
- Container `push` = bundle-export out + `hostSidePush` from a fresh host
  edge (validated captured origin, empty-helper reset, token host-side) —
  "deliberately kind-shaped for the remote kind": `files.read` (binary)
  replaces `docker cp`.
- Capture: `makeClensHooks`'s `container.fetchTranscript` is a generic
  `(containerPath) => string | undefined` callback — a sandbox `files.read`
  fetch slots in.
- `cli.ts:158` explicitly refuses `--isolation remote` today.

## Operator decisions (2026-07-18)

1. **Plan §5 amendment #6 (approved spike-gated):** `AgentSpawnSpec`
   becomes a discriminated union — `{kind:"container", containerId,
   workdir} | {kind:"e2b", sandboxId, workdir}` — and `live-query.ts` maps
   the e2b variant onto `spawnClaudeCodeProcess` via an E2B-backed
   `SpawnedProcess` adapter (background `claude` command with
   `stdin: true`; stdin → `sendStdin`, stdout → `onStdout`, kill →
   `commands.kill(pid)`). Amendment text lands in `specs/adw-v1-plan.md` §5
   BEFORE implementation. Env crosses via the command's `envs`, never argv
   (the M4 ps-visibility rule).
2. **Keep = pause (Art. VII, honest), FILESYSTEM-ONLY:** a blocked/aborted
   remote run's sandbox is PAUSED (state preserved indefinitely, billing
   stops) — not left to expire ("retention window" wording replaced), not
   killed. **The pause MUST be filesystem-only (`keepMemory: false`)**
   (Codex 2026-07-18, High): the default full-memory snapshot captures the
   running claude process's RAM, which holds the `CLAUDE_CODE_OAUTH_TOKEN`
   supplied in its env — a memory snapshot would persist the token at rest,
   defeating the token-at-rest guarantee that env/fs scans check. A
   filesystem-only snapshot cold-boots on resume (the running process is
   lost — fine, we need only the on-disk checkout + SDK session file). Crash
   safety uses the SDK's real field — **`lifecycle: {onTimeout: {action:
   "pause", keepMemory: false}}`** (NOT `autoPause: true`, which is not a
   `SandboxOpts` field in `e2b@2.35.0` and would be silently ignored /
   type-rejected — Codex High, confirmed against the installed types) — so
   even a crashed factory's sandbox pauses filesystem-only at timeout.
   `adw clean` reclaims with static **`Sandbox.kill(sandboxId)`** (no
   reconnect — reconnecting would resume a snapshot before deletion). An
   OPEN m5-02 probe: prove SDK-session `query({resume})` still works after a
   `keepMemory:false` cold boot (m5-01 proved the FILESYSTEM survives resume
   with the DEFAULT full-memory pause — cold-boot session resume is NOT yet
   verified; do it before committing this path).
3. **Clean wiring in scope; CI round deferred:** marker + breadcrumb +
   `adw clean` land HERE (a kept sandbox must have a reclaimer from day
   one). The remote CI maintenance round is **adw-m5-03** (queued, coarse);
   until it lands, a remote attempt's CI round is an explicitly reported
   limitation, the m4-04 precedent.
4. **Executable-integrity on E2B (Codex Critical) — CONFIRMED live, and
   RESOLVED with a proven fix (2026-07-18 gate-integrity spike).** E2B's
   post-layer provisioning `chmod`s `/usr/local` world-writable, and worse:
   `bun` is a 777 **regular file** at `/usr/local/bin/bun`, so the agent
   truncates it directly (`printf x > …/bun` → `bun --version` prints
   `FORGED-GATE`) — no shadowing needed. `git` (`/usr/bin`), `sh` (`/bin`),
   `/opt` are all root-owned + locked; only `/usr/local/bin` is exposed.
   Restricting the gate PATH via `commands.run` `envs` does NOT help (the
   login shell resets PATH). **The fix (spike-verified):** immediately after
   `Sandbox.create`, the factory runs ONE privileged host-orchestrated
   `commands.run("chmod -R go-w /usr/local", {user: "root"})` — E2B's default
   `user` has passwordless sudo, so this works without a template rebuild.
   Verified post-relock: `bun` is 755 root-owned again (real 99 MB binary),
   overwriting it → `Permission denied`, planting a new file in the now-755
   dir → blocked, `git`/`claude` intact. This RESTORES the container-kind's
   root-owned-immutable-PATH invariant (m4-05 H2 parity). Belt-and-suspenders:
   the spawn adapter still invokes the immutable
   `/opt/claude/share/versions/<ver>/claude` absolute path. A live negative
   test (post-relock tamper attempt) is a hard requirement.

## Requirements

- [x] **`src/workspace/e2b.ts` — provision:** `Sandbox.create(alias,
      {timeoutMs, metadata: {adwRunId, adwTicketId}, lifecycle: {onTimeout:
      {action: "pause", keepMemory: false}}})` from the m5-01 template alias
      (decision 2 — NOT `autoPause`; metadata per the create-atomicity
      decision below); clone into the fixed sandbox workdir **`E2B_WORKDIR =
      "/work/remote"`** — bun-writable and DISTINCT from `CONTAINER_WORKDIR`
      (`/work/repo`, the container marker). NOT `/home/user/repo`: m5-01
      proved `/home/user` belongs to E2B's separate `user`, so a `bun`-user
      clone there fails (Codex High, confirmed). Bundle pattern: host `git
      bundle create` at the resolved branch point → binary `files.write` →
      in-sandbox `git init && git fetch <bundle>` on the attempt branch.
      Origin captured from the HOST repo + `validateOriginUrl` at provision
      (`captureValidatedOrigin`, the m4-07 trust rule). Branch collisions
      consulted against the host repo (m2-01). Reclaim breadcrumb
      `{kind:"e2b", sandboxId, branch, ticketId}` written to
      `runs/<runId>/workspace.json` the moment the sandbox exists. Failures
      throw `WorkspaceError` (ticketId + node, E5).
- [x] **Create atomicity / orphan reconciliation (Codex Medium) — SPLIT
      across phases (explicit, not silent; validator Finding 3 / amendment
      rule):** `Sandbox.create` and the local breadcrumb write are not atomic
      — a lost create response or a failed breadcrumb write orphans a running
      sandbox, and a naive retry duplicates it. The ENABLING half lands in
      Phase 2 (done): `metadata: {adwRunId, adwTicketId}` is stamped on create,
      and the breadcrumb is written the instant the sandbox exists (before the
      relock), so a failed-breadcrumb-write path is covered and the
      breadcrumb-before-relock ordering is tested. The CONSUMING half — the
      `Sandbox.list()`-by-`adwRunId` metadata DISCOVERY that reclaims a
      lost-create-response orphan (a sandbox that started but whose id never
      reached the factory) — lands in **Phase 4 (clean wiring)**, where the
      `list` edge naturally belongs alongside the e2b clean path; provision's
      pre-retry metadata query rides that same `list` capability. This is a
      within-ticket phase ordering, not a scope cut: the orphan IS reclaimable
      once Phase 4 lands. The Phase-4 fake-edge tests cover the lost-response
      discovery path.
- [x] **exec:** foreground `commands.run(cmd, {cwd: E2B_WORKDIR, envs,
      timeoutMs})` → `ExecResult`; the plan §5 amendment-#5 kill ceiling
      applies (`DEFAULT_EXEC_TIMEOUT_MS` default — E2B's own 60s default
      MUST be overridden or long gates die); a timeout returns non-zero
      naming the timeout, never throws.
- [x] **agentEnv:** `ADW_TICKET_ID` + `CLAUDE_CODE_OAUTH_TOKEN` (M4
      pattern; TRACEPARENT stays per-node engine merge, S5.3). The
      sandbox-env at rest carries NO credential; the token exists only in
      the agent command's env at exec time. `E2B_API_KEY` NEVER enters the
      sandbox in any form.
- [x] **agentSpawn (amendment #6):** workspace exposes
      `{kind:"e2b", sandboxId, workdir: E2B_WORKDIR}`; `live-query.ts`
      gains the e2b mapping per decision 1 — pure spawn-policy helpers
      unit-tested, the adapter tested against a fake command handle (the
      `ContainerSpawnImpl` recorder precedent), the SDK's forwarded
      teardown signal honored (m4-06). The spawned command MUST be the
      immutable **`/opt/claude/share/versions/<ver>/claude`** absolute path,
      never `claude` off the writable PATH (decision 4, claude-integrity).
      m5-01 proved the graceful path exists: `CommandHandle.closeStdin()`
      delivers real stdin-EOF, so the SDK's stdin-close teardown works
      (no hard-kill-only compromise); `hardStop` still backstops via
      `commands.kill`.
- [x] **push:** export the gate-green commit as a bundle via in-sandbox
      `git bundle create` + binary `files.read`, then publish through the
      existing host-side edge (shared structure with `hostSidePush` —
      `hostPushArgv`, empty-helper reset, provision-captured origin,
      per-push minted token host-side only; Art. VIII: extract the shared
      edge rather than duplicating it).
- [x] **hardStop:** kill the in-sandbox agent process(es)
      (`commands.kill`) WITHOUT killing the sandbox — state kept for
      autopsy (Art. V + VII), mirroring "container left EXITED". Spike
      facts from m5-01 decide the exact mechanism; document it.
- [x] **keep = pause / teardown:** teardown (clean-only) = static
      **`Sandbox.kill(sandboxId)`** (NOT connect-then-kill — reconnecting
      resumes the snapshot before deletion; Codex High), tolerant of an
      already-gone sandbox. Blocked/abort paths leave the sandbox to pause
      filesystem-only via the `lifecycle.onTimeout` policy; an explicit
      `pause({keepMemory:false})` on the finalize path only if a probe shows
      the lifecycle policy is unreliable. Grep-level teardown exclusivity
      holds (clean stays the only caller).
- [x] **clean wiring:** `parseWorkspaceBreadcrumb` widens to the e2b kind;
      `keptWorkspace` + the attempts probe recognize `E2B_WORKDIR` and
      probe sandbox existence via an injected E2B edge (paused counts as
      existing); breadcrumb reclaim path covers e2b. Worktree + container
      behavior byte-identical.
- [~] **capture parity — DEFERRED to the live bar / a follow-on (explicit,
      not silent; amendment rule):** two real seam mismatches surfaced in
      Phase 5 that make e2b `fetchTranscript` more than a mechanical mirror
      of m4-03: (1) the `makeClensHooks` `container.fetchTranscript` seam is
      SYNC (`(path) => string | undefined`, container backs it with
      `execFileSync docker cp`), but E2B's `files.read` is async-only — no
      sync-over-network path; (2) the container binding keys on
      `containerName(runId)` (deterministic), but a sandboxId is
      SERVER-ASSIGNED and known only post-provision, so the fetch must
      resolve it from the run breadcrumb at call time. Closing this cleanly
      needs either a sync helper subprocess (a `bun` child that connects +
      reads, execFileSync'd — the docker-cp analog) or an async capture seam.
      The remote lane RUNS without it (provision → agent → gates → push →
      PR); only the in-sandbox cLens transcript is not host-copied yet. This
      mirrors M4, where container capture was a post-landing live finding
      (m4-03). Wire + verify at the live bar or file a small follow-on;
      tracked, not dropped.
- [x] **Provision hardening (decision 4, the gate-integrity fix):** right
      after `Sandbox.create`, before ANY agent/gate command, the factory
      runs `commands.run("chmod -R go-w /usr/local", {user: "root"})` to
      relock the E2B-777'd trusted-binary dir (spike-verified). A failed
      relock throws `WorkspaceError` (provision fails loud, no agent runs).
- [x] **CLI:** `--isolation remote` accepted; pre-flight refusals for a
      remote dispatch missing `E2B_API_KEY` or `CLAUDE_CODE_OAUTH_TOKEN`
      (the M4 container-auth shape); remote CI-round attempts report the
      adw-m5-03 limitation honestly (skip/state-error shape, no charge —
      exact wording pinned in tests).
- [~] **Executable-integrity live negative test (Codex Critical):**
      post-relock, a `bun`-user attempt to overwrite `/usr/local/bin/bun`
      AND to swap `/usr/local/bin/claude` both fail (`Permission denied`),
      and the real `bun`/`claude` still run — proving both gate-integrity
      (the relock) and claude-integrity (immutable-path spawn) hold live.
- [x] **Contract suite reuse UNCHANGED** (`test/workspace/contract.ts`,
      S4.3): an e2b `WorkspaceContractFactory` runs the same suite, gated
      like the docker suite (skip without `E2B_API_KEY` / network),
      lean-budget compliant (short timeouts, killed on teardown).
- [x] **Live bar (scratch repo):** full-lane toy chore under
      `--isolation remote` → PR opens; remote agent spans parent under the
      run trace (S5.3, epic exit); token-at-rest scans (sandbox env +
      fs: no E2B key, no OAuth token, no git token); blocked/abort probe →
      sandbox paused, `adw clean` reclaims it; a negative control on the
      push path (the m4-07 lesson: ask what the green suite structurally
      cannot see).

## Closeout (2026-07-19, operator-approved flip → done)

Live bar **MET**: a real cLens chore (`clens-006-distill-io-seam`) completed the
FULL lane under `--isolation remote` on E2B — runId
`clens-006-distill-io-seam-1784470261685` (`runs/…`, journal
`run-end outcome=green durationMs=515329`, sandbox `ie88qac4yg2typbo024q1`),
dispatch → provision → build → gates (lint/typecheck/test all pass) → commit →
push → open-pr, isolation=remote/kind=e2b, PR opened on `adw/clens-006-…`.
Offline suite green (634 tests, tsc+lint). Honest carry-overs:

- **Box 203 `[~]`** — executable-integrity live negative: spike-verified at
  build (commit `ee96043` folded a live negative into provision as a hardening
  step; `E2B_RELOCK_COMMAND` unit-asserted). No *standing automated* live
  negative test in the suite — recorded as a known gap, not a blocker.
- **Box 212 sub-probes** — the green run proves full-lane + PR + remote agent
  spans under the run trace (S5.3). The extra live probes (token-at-rest env/fs
  scans, blocked→pause→`adw clean` reclaim, push negative control) were exercised
  across the M5-04/05 live investigations, not re-run in this single green pass.

Flipped to `done` per operator decision 2026-07-19 (I hold no live keys to re-run
the billed bars). adw-m5-03's resumed-repair live bar remains OPEN (that ticket).

## Plan §5 amendment (to land, operator-approved 2026-07-18; spike PASSED)

`AgentSpawnSpec` → discriminated union (`container` | `e2b {sandboxId,
workdir}`); live-query maps e2b onto `spawnClaudeCodeProcess` via the E2B
`SpawnedProcess` adapter. The m5-01 spike PASSED the gate (adw-m5-01
Result): `query()` over the E2B transport returned `is_error=false`,
`closeStdin()` gives real graceful stdin-EOF, binary round-trip + pause/
resume hold. Wording lands in `specs/adw-v1-plan.md` §5 before any source,
citing this ticket + the m5-01 spike evidence. The adapter spawns the
immutable `/opt/claude/share/versions/<ver>/claude` absolute path
(decision 4).

## Build protocol

Red-first at these seams (validator-gated per the build protocol):

1. **types.ts + plan §5:** amendment text; union type; existing container
   consumers untouched (compile-level proof).
2. **e2b.ts:** injected E2B client edges (sandbox/commands/files fakes in
   tests — the CiContainerOps precedent); pure helpers (argv/env
   assembly, breadcrumb, marker) unit-tested; the real-SDK calls confined
   to thin edges; e2b-gated live contract suite.
3. **live-query.ts:** e2b spawn mapping (pure policy + fake-handle
   adapter tests).
4. **clean.ts:** breadcrumb union + marker recognition + injected
   existence probe.
5. **cli.ts:** isolation guard, preflights, capture binding.
6. Full suite + lint + tsc; two-axis /code-review vs the pre-ticket base;
   live bar per the requirement above (lean budget).

## Out of scope

Remote CI round (adw-m5-03); parallel runs; ssh push (adw-m4-08).

## Verify

Contract suite green against the remote kind; full-lane toy chore under
`--isolation remote`; `bun run lint && bunx tsc --noEmit && bun test`
green.
