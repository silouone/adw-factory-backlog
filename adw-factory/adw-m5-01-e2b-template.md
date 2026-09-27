---
id: adw-m5-01-e2b-template
type: chore
status: done
priority: 3
created: 2026-07-14
epic: adw-m5
depends: [adw-m4]
attempts: []
---
# E2B custom template

> Refined at pickup 2026-07-18 (plan §8, protocol §3) — operator-approved
> decisions recorded below. Grounded in the E2B docs current at refinement
> (JS SDK v2.29.x): templates are defined IN CODE via the Template SDK
> (`Template().from…()` fluent builder, `Template.build(template, alias)`)
> and cloud-built — no local docker in the build path. `E2B_API_KEY` is the
> SDK's own default env var.

## Context

The remote analog of `containers/Dockerfile` (adw-m4-01): a committed,
reviewable template baking everything the in-sandbox agent needs — bun,
git, the pinned Claude Code CLI — and NOTHING credential-shaped (N1).
Auth arrives only as process env at exec time (M4 pattern:
`CLAUDE_CODE_OAUTH_TOKEN` via `agentEnv()`); the E2B key stays HOST-side
only and never enters the sandbox. Cites S4.5, N1.

This ticket's live verify doubles as the **spike gating plan §5 amendment
#6** (`AgentSpawnSpec` union + e2b live-query mapping, operator-approved
2026-07-18 contingent on the spike): the Agent SDK's `SpawnedProcess` is a
duck-typed ChildProcess, and E2B's `commands.run(cmd, {background: true,
stdin: true})` + `commands.sendStdin(pid, …)` + `commands.kill(pid)` look
sufficient to satisfy it — the spike proves or refutes that on a real
sandbox before adw-m5-02 writes any amendment text or source.

## Operator decisions (2026-07-18)

1. **E2B key from `E2B_API_KEY` only** (the SDK default) — env at build/run
   invocation time, never committed, never logged, never forwarded into a
   sandbox.
2. **Lean budget** (free/low tier): short sandbox `timeoutMs` for every
   probe, explicit `kill()` the moment a probe batch ends, batch live
   proofs — no idle sandboxes.
3. **§5 amendment #6 approved spike-gated**: if this ticket's spike shows
   the adapter cannot work (stdin/EOF semantics, Bun-host SDK compat), STOP
   and re-propose before adw-m5-02.

## Deliverables

- `containers/e2b-template.ts` — committed template definition (Template
  SDK). Preferred: reuse `containers/Dockerfile` via `fromDockerfile()`
  so the two isolation kinds share ONE image contract (Art. VIII);
  fallback (E2B's Dockerfile parser rejecting a directive, e.g. `ARG`): a
  fluent-builder mirror of the same pinned steps, with the divergence
  documented in the module header. Exports the template + the alias
  constant (`adw-agent`) adw-m5-02 will reference.
- `scripts/e2b-template-build.ts` — build entry (`Template.build`, alias
  `adw-agent`), printing name/templateId/buildId; fails fast when
  `E2B_API_KEY` is absent, and never prints the key.
- Spike record in this ticket's Result section (throwaway spike code under
  the session scratchpad — not committed source; Art. II).

## Requirements

- [~] **Template parity with adw-m4-01 — PARTIAL, one gap confirmed:** bun
      (pinned base), git, Claude Code pinned `2.1.210` installed ROOT-owned
      at `/opt/claude` (met — `/opt` is untouched by E2B provisioning);
      NO `gh` (met); zero secrets baked (met). **NOT met — the H2 "cannot
      replace OR shadow" guarantee does NOT carry to E2B** (Codex Critical,
      live-confirmed): E2B's post-layer provisioning sets `/usr/local/bin`
      to 777, so the agent can replace the `claude` SYMLINK there and shadow
      trusted binaries. Not fixable at template-build time (E2B re-777s at
      provision, after our layers). Deferred to adw-m5-02: claude-integrity
      via absolute immutable-path spawn; gate-integrity is an open
      operator-gated question. See Result + adw-m5-02 decision 4.
- [x] **Committed + reproducible:** template defined in committed code;
      every version pinned; the alias↔claude-version convention documented
      in the module header (the m4-01 tag convention, carried over).
- [x] **Build:** `bun scripts/e2b-template-build.ts` builds the template in
      the E2B cloud from `E2B_API_KEY` env only; templateId/buildId
      recorded in this ticket.
- [x] **Sandbox smoke:** a sandbox created from the alias runs
      `bun --version`, `git --version`, `claude --version` (pinned value)
      — note which USER commands run as (E2B default vs image user; the
      root-owned claude protection must hold for the actual exec user).
- [x] **Agent-call spike (the amendment gate):** a trivial agent call
      completes with claude running IN the sandbox on subscription auth
      (`CLAUDE_CODE_OAUTH_TOKEN` passed as command env at exec time only),
      driven from the host through a prototype `SpawnedProcess` adapter
      over `commands.run({background: true, stdin: true})` +
      `sendStdin`/`onStdout`. The spike must answer, with evidence:
      (a) does the SDK protocol handshake survive the E2B transport;
      (b) is there a workable stdin-EOF / graceful-stop path (the SDK's
      teardown is stdin-close → grace → kill), or does hard-kill-only
      apply (document which — the m4-06 precedent);
      (c) does the `e2b` JS SDK run under the factory's Bun host;
      (d) binary `files.read` of an in-sandbox file round-trips intact
      (the future bundle-export path for `push`);
      (e) `autoPause`/`pause()` + `Sandbox.connect()` preserves the
      filesystem across resume (the pause-on-keep bet of adw-m5-02).
- [x] **Secret scan analog (N1):** in-sandbox `env` shows no *_KEY /
      *_TOKEN / *_SECRET at rest (before any agent exec); the template
      definition greps clean of token patterns (via `findBakedSecrets` —
      broadened after the Codex finding that the original inline regex
      missed `CLAUDE_CODE_OAUTH_TOKEN`, fixed red-first, commit 06eab06);
      `E2B_API_KEY` provably never enters sandbox env.
- [x] **Lean discipline:** every spike sandbox created with a short
      `timeoutMs` and explicitly killed when its batch ends.

## Build protocol

Chore (protocol §3): no pipeline source. Adds the `e2b` npm dependency
(pinned). TDD Art. I applies to any TS with logic — the template module is
declarative data; if a pure helper emerges (e.g. env/step assembly), it
gets a red-first unit test. Spike code stays out of the repo; its FACTS
land in the Result section here. Live E2B usage follows the lean-budget
decision above.

## Verify

Template builds; a sandbox from it runs `bun`, `git`, `claude` and
completes a trivial agent call on subscription auth; secret-scan analog
clean; `bun run lint && bunx tsc --noEmit && bun test` untouched-green.

## Result (2026-07-18)

Live half complete; 519 tests, lint + tsc green. Red set validator-APPROVED
before source. Build: **43s cloud build**, `adw-agent` tags `[2.1.210]`,
templateId `z5e3w17rtltrzz8ea9bf`, buildId
`b782fe3d-da6b-4983-9629-c7eacd81127a`. `fromDockerfile` accepted the whole
pre-resolved Dockerfile — FROM/RUN/USER/ENV/WORKDIR all honored (exec user
IS `bun`, default cwd IS `/work`).

**Smoke (sandbox from `adw-agent:2.1.210`):** bun 1.2.4, git 2.30.2,
claude 2.1.210; claude root-owned at `/opt/claude` → `/usr/local/bin`
symlink; `bun` has NO sudo (E2B grants passwordless sudo only to its
default `user`). In-sandbox `env` scan CLEAN (no *_KEY/_TOKEN/_SECRET);
E2B_API_KEY provably absent.

**Spike (the §5 amendment-#6 gate) — PASSED, all five questions:**
(a) SDK protocol survives the E2B transport: a real `query()` with
`spawnClaudeCodeProcess` mapped onto `commands.run(claude …, {background,
stdin: true})` returned `E2B-SPIKE-OK`, `is_error=false`.
(b) Graceful stop EXISTS: `CommandHandle.closeStdin()` delivers real
stdin-EOF — the agent exited code 0 with no kill (BETTER than the container
kind, which needs the hard-kill backstop; envd-version-gated, fine on a
freshly built template).
(c) `e2b@2.35.0` runs under the factory's Bun host (1.2.4).
(d) Binary `files.write`/`files.read({format:"bytes"})` round-trips 64 KiB
sha256-identical both ways (the m5-02 bundle path).
(e) `pause()` → `Sandbox.connect()` preserves the filesystem (marker file
survives resume) — pause-on-keep holds.
Token-at-rest post-agent: a fresh exec sees no CLAUDE_CODE_OAUTH_TOKEN
(command-scoped envs never persist).

**Security finding (recorded, not hidden):** E2B provisioning runs AFTER
the template layers and sets `/usr/local` to 777 — a write probe as the
agent user in `/usr/local/bin` succeeds, so the H2 "cannot overwrite OR
shadow claude" guarantee does NOT fully carry to E2B: `/opt/claude` stays
root-owned but the SYMLINK is replaceable in-sandbox. Exposure: an agent
could plant a trojan `claude` that a LATER spawn in the same sandbox (CI
round resume) would exec with a fresh token. Mitigation candidates for
adw-m5-02 (probe there): template `setStartCmd` chmod (if it runs as
root), or spawn via the immutable `/opt/claude/share/versions/<v>` real
path instead of the symlink. The sandbox boundary itself (vs the operator
host) is unaffected.

**Spike-probe corrections worth remembering:** (1) a spike file OUTSIDE
the repo made Bun auto-install agent-sdk 0.3.214 (which host-checks cwd
and broke the container-style spawn) instead of the repo's pinned 0.3.209
— run repo-coupled spikes from inside the repo; (2) `/home/user` is owned
by E2B's `user`, NOT the image's `bun` — the m5-02 remote workdir must be
bun-writable and distinct from `/work/repo` (the container marker):
decision for m5-02, e.g. `/work/remote`.

Lean: one build + two short-lived sandboxes, all killed; zero left
running. Cost: ~5 sandbox-minutes total.

## Codex adversarial review — disposition (2026-07-18)

Operator ran `/codex:adversarial-review` over the M5 span (base ad470f7 →
a984934). Verdict needs-attention: 1 Critical, 3 High, 2 Medium. Each
challenged against the installed `e2b@2.35.0` types and the m5-01 live
spike (the m6-03 method). NONE were false-accepts.

- **[Critical] Writable PATH — CONFIRMED** by my own spike (E2B 777s
  `/usr/local/bin`; the `claude` symlink is agent-replaceable). Falsifies
  this ticket's template-parity H2 box (re-stated `[~]` above). Two halves:
  claude-integrity (absolute immutable-path spawn — clean, m5-02) and
  gate-integrity (fake `bun`/`git` forging gates — load-bearing, no proven
  fix, OPERATOR-GATED before m5-02 ships). Disposition of a confirmed
  Critical is the operator's call → this ticket is NOT self-flipped to done.
- **[High] Full-memory pause persists the OAuth token — VALID** (default
  `keepMemory:true` snapshots the running agent's RAM incl. the env token).
  m5-02 decision 2 now mandates `keepMemory:false` (filesystem-only) +
  static `Sandbox.kill`. My spike proved fs-survives-resume only with the
  DEFAULT full-memory pause; cold-boot session resume is an m5-02 probe.
- **[High] `autoPause` ignored by 2.35.0 — CONFIRMED against the types**
  (`SandboxOpts` has `lifecycle.onTimeout`, no `autoPause`). Spec-text-only
  correction (no shipped code created a sandbox — lower than Codex's High):
  m5-02 now uses `lifecycle: {onTimeout: {action:"pause", keepMemory:false}}`.
- **[High] Workdir `/home/user/repo` not bun-writable — CONFIRMED** by my
  spike (that path is E2B `user`'s). m5-02 workdir → `/work/remote`.
- **[Medium] Create/breadcrumb non-atomic → orphan — VALID.** m5-02 gains
  create `metadata:{adwRunId}` + `Sandbox.list` reconciliation in clean.
- **[Medium] N1 scan misses `CLAUDE_CODE_OAUTH_TOKEN` — CONFIRMED, FIXED
  red-first HERE** (findBakedSecrets, 06eab06): the one review defect in
  already-committed m5-01 code, now closed with negative-control fixtures.

Net: one m5-01 code defect fixed (N1); five findings are m5-02 spec
refinements (already folded into that ticket, still queued). The Critical's
gate-integrity half + whether m5-01 counts as done despite it → operator.

**Operator disposition (2026-07-18):** m5-01 DONE — template+spike deliverable complete; the confirmed Critical is exploitable only via m5-02 lane (not yet built) and its fix is fully speced into adw-m5-02 (claude-integrity solved there, gate-integrity spiked-then-decided per operator). Gate-integrity approach: m5-02 spike scopes the real 777-PATH exposure and tests a root start-cmd relock, operator decides disposition before ship.
