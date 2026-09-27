---
id: adw-m4-02-container-workspace
type: feat
status: done
priority: 3
created: 2026-07-14
epic: adw-m4
depends: [adw-m4-01-container-image]
attempts: []
---
# Container Workspace implementation

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`. Contains a REQUIRED design spike — the
> agent-in-container execution seam was never analyzed before this
> refinement.

## Context

Second `Workspace` kind behind the exact interface proven in M1; running
`test/workspace/contract.ts` UNCHANGED against it is the proof of S4.3.
**Refinement finding:** under worktree isolation the agent process runs on
the HOST (SDK `query()` spawns claude locally with `cwd = workspace.path`).
A container workspace is only "fully contained" if claude runs INSIDE the
container — which is why adw-m4-01 bakes it into the image. The Agent SDK
talks to its claude subprocess over stdio, so the candidate seam is an
executable shim (`docker exec -i -w <path> <ctr> claude "$@"` via the
SDK's executable-path option, or an equivalent per-kind query seam in
live-query.ts). Wrinkles to resolve in the spike: the host-side spawn cwd
(`workspace.path` is an in-container path that does not exist on the
host), env delivery through `docker exec`, and stream fidelity
(init/result messages intact). **Amendment rule:** if the spike shows the
`Workspace` contract needs a new member (e.g. the workspace supplying its
agent spawn command), STOP and propose the spec/plan amendment before any
implementation.

## Deliverables

- Spike findings written into this body (mechanism chosen + rejected)
- `test/workspace/container.contract.test.ts` — the shared suite bound to
  a container factory (first, red) + kind-specific provision tests
- `src/workspace/container.ts`
- `src/cli.ts`: `--isolation container` accepted (remote still refused,
  S4.5); `src/clean.ts`: reclaim path for kept containers (S4.4)
- `src/live-query.ts` (or equivalent seam): in-container agent execution

## Requirements

- [x] **Spike (before red tests are finalized):** prove one SDK `query()`
      drives claude inside a running M4-image container end-to-end
      (init + result messages received); document the seam here
      (done 2026-07-16 — see "Spike findings" below)
- [x] `provisionContainer`: start a container from the pinned M4 image;
      clone the target repo inside it on a fresh attempt branch — branch
      naming reuses `nextAttemptBranch` consulted against the HOST repo's
      local+remote branches (collision scope, adw-m2-01 finding); run
      `target.setup` in-container; throw typed `WorkspaceError` before any
      agent runs on failure (E5)
- [x] `Workspace` members: `kind: "container"`; `path` = the in-container
      checkout (contract's `pwd -P` probe must pass through `exec`);
      `exec` = `docker exec` sh -c at `path`; `agentEnv()` =
      `ADW_TICKET_ID` (+ the m4-03 auth slots); `teardown()` = remove
      container + host bookkeeping — invoked ONLY by `adw clean`;
      blocked/abort keeps the container for autopsy (S4.4, Art. VII)
- [x] `describeWorkspaceContract` runs UNCHANGED against the kind (S4.3);
      the suite skips cleanly (with a loud notice) when docker is absent,
      but this ticket's verify runs it for real
- [x] Pipeline untouched: no lane/node logic changes beyond injected seams
      (S4.3, S4.5); the sync-pr-state negative-capability grep guard scans
      container.ts too — it must stay merge-incapable (Art. IV)
- [x] Observability parity: journal + spans + capture flow identically;
      TRACEPARENT rides the same ctx merge (adw-m3-03 — this is exactly
      what that design bought, S5.3)

## Spike findings (2026-07-16)

**Mechanism chosen: SDK `options.spawnClaudeCodeProcess`** — the Agent SDK
(0.3.x) exposes a first-class custom-spawn hook documented for exactly this
("Use this to run Claude Code in VMs, containers, or remote environments");
Node's `ChildProcess` already satisfies its `SpawnedProcess` interface.
Proven end-to-end against a running `adw-agent:2.1.210` container (repo at
`/work/repo`): one `query()` with a custom spawn of
`docker exec -i -w /work/repo -e … <ctr> claude <sdk args>` produced the
`system:init` message (session id, **in-container** `cwd:/work/repo`, model)
and a `result` message through docker-exec stdio, stream intact. The result
was `Not logged in` — auth is adw-m4-03; everything up to auth is proven.

Empirical facts the implementation can rely on:

- `SpawnOptions.args` are pure protocol flags (`--output-format stream-json
  --verbose --input-format stream-json --model … --permission-mode …`) — no
  host paths; `SpawnOptions.command` (host CLI path) is safely discarded and
  replaced by in-container `claude`.
- On the custom-spawn path the SDK does NOT require `cwd` to exist on the
  host — the in-container path rides through untouched (the refinement's
  "host-side spawn cwd" wrinkle dissolves).
- `SpawnOptions.env` is minimal: exactly our `options.env` plus the SDK's
  own `CLAUDE_CODE_ENTRYPOINT` + `CLAUDE_AGENT_SDK_VERSION` — NOT the host
  process env. Forwarding only these via `docker exec -e` gives the strict
  env allowlist the adw-m6-03 review deferred to M4; host PATH/HOME never
  cross the boundary.
- Host-SDK 0.3.211 driving in-container CLI 2.1.210: protocol-compatible
  (observed; both pins recorded).
- **Kill semantics (verified):** stdin-EOF graceful shutdown works through
  `docker exec -i` (the SDK's primary close path). But SIGKILLing the host
  docker-exec client does NOT kill the in-container process — it leaks until
  container-level action. Real OS-level kill (the M6 watch item) is
  `docker kill`/`docker rm -f` — i.e. teardown/`adw clean`; blocked/abort
  keeps the container (Art. VII), so a wedged agent process can persist in a
  kept container. Documented as a container.ts concern.

**Rejected:**

- *Executable shim script* (`pathToClaudeCodeExecutable` → generated
  `docker exec` wrapper): works in principle but strictly worse — an extra
  generated artifact per run, same kill semantics, and the SDK's
  spawn-hook is the documented seam for containers.
- *env side-channel* (`ADW_CONTAINER_ID` in `agentEnv()`, sniffed by
  live-query): no contract change, but a covert control channel that muddies
  the exact-env contract tests m4-03 extends. Dishonest seam.
- *Constructing the query after provision* (per-run query close-over): the
  `AgentQuery` is injected at lane construction, provision runs mid-lane —
  restructuring that wiring is a pipeline change, exactly what S4.3 forbids.

**Contract gap → amendment required (the anticipated case).** `AgentQuery`
is injected before any workspace exists; the workspace reaches agent nodes
only as `cwd` + `agentEnv()`. live-query has no channel to learn "spawn this
one inside container X". Per the amendment rule, STOPPED here and proposed
(see below) before any implementation.

## Amendment proposal (plan §5 Workspace contract) — APPROVED by operator 2026-07-16; landed in plan §5 + types.ts

Add ONE optional, data-only member to `Workspace`:

```ts
interface Workspace {
  // …existing members unchanged…
  /** How agent processes are spawned for this kind;
   *  absent → the SDK's default local spawn (worktree). */
  readonly agentSpawn?: AgentSpawnSpec
}
/** Pure descriptor; mapped onto the SDK's spawnClaudeCodeProcess by
 *  live-query.ts (the one real-SDK file — now also the one docker-exec-
 *  for-agent edge). */
interface AgentSpawnSpec {
  readonly kind: 'container'
  readonly containerId: string
  readonly workdir: string   // in-container agent cwd (== workspace.path)
}
```

Ripples ride the already-injected seam, not lane logic: `AgentWorkspace`
(build.ts structural subset) and `AgentQueryOptions` gain the same optional
field; `baseOptions` forwards it only when present — a worktree run's
options stay **byte-identical** to today (the M3 hooks/traceparent
precedent). Data-only keeps container.ts pure/testable without docker
(Art. IX); the docker-exec spawn I/O stays in live-query.ts.

## Build protocol (Art. I)

1. Spike (throwaway, no factory source). 2. Red tests: contract suite
   bound to the kind + provision/branch/teardown specifics + cli guard
   narrowing. Validator review per protocol. 3. Implement to green.

## Verify

Contract suite green against the container kind (docker present);
`bun run lint && bunx tsc --noEmit && bun test` green; full-lane toy chore
under `--isolation container` with the FAKE query (integration test). The
LIVE full-lane container run is adw-m4-03's verify — it needs in-container
auth.

**Verified 2026-07-16 (real docker, OrbStack):** contract suite 11/11
against the container kind (`test/workspace/contract.ts` byte-unchanged);
full-lane integration green — dispatch → … → open-pr under
`--isolation container`, fake-agent edit made INSIDE the container via the
delivered agentSpawn descriptor, attempt branch pushed from in-container git
onto the host origin, journal `run-start` carrying `container`, container
kept after green and after blocked, `adw clean` (kind-aware `keptWorkspace`)
reclaiming it; blocked path records `{branch, workspace:/work/repo}` read
back from the kept container. Red set validator-APPROVED before source
(commit 477f168); full suite 424 pass / lint / tsc green.

Implementation notes (facts a future reader needs):

- The clone travels as a `git bundle` of exactly the branch point (host →
  `docker cp` → in-container `git fetch <bundle> '<ref>:refs/heads/<branch>'`)
  — no network, no host-source mount. A LOCAL-path origin is the one
  deliberate mount (`-v <origin>:/adw/origin`, rw) so the push node's
  in-container `git push` can land on it; URL origins cross as URLs (auth =
  adw-m4-03). Repo-local git identity is set at provision (headless
  containers have no gitconfig; the commit node needs it).
- Container teardown fails fast on an already-gone container (docker's
  `rm -f` went idempotent; an inspect-first check preserves worktree-kind
  parity — double teardown = bookkeeping desync, Art. IX).
- `agentSdkEnv` (live-query): the container path hands the SDK ONLY the
  workspace/options env — the strict allowlist adw-m6-03 deferred to M4;
  host PATH/HOME/secrets never cross the boundary.
- Kind detection for recorded attempts (clean, finalizer) is the fixed
  `CONTAINER_WORKDIR == /work/repo` invariant from the image contract;
  container existence rides an injected probe (`CleanDeps.containerExists`,
  live docker default).
- Known follow-up (in scope for adw-m4-03's live lane if hit): the CI
  mini-lane (`ci-round.ts`) still reattaches kept WORKTREES only — a
  container-run ticket entering a CI round is not yet supported; scoped out
  here deliberately (this ticket's lane ends at open-pr).

## Out of scope

Auth of any kind (adw-m4-03); remote/E2B (adw-m5); parallel runs.
