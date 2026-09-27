---
id: adw-m4-07-host-side-push
type: feat
status: done
priority: 3
created: 2026-07-18
epic: adw-m4
depends: [adw-m4-05-review-fixes]
attempts: []
---
# Trusted host-side push boundary for container isolation

> Spec (SDD). Written 2026-07-18 from the second Codex adversarial round on
> M4 (thread 019f73d3). Carries full requirements — spec-complete, no
> refinement gate; awaits operator go-ahead before `in-progress`. Cites
> `S<n>` = adw-v1.md §3 · `N<n>` = §4 · `Art.<n>` = constitution.

## Problem Statement

When the operator runs a ticket under `--isolation container`, the factory
performs the authenticated `git push` **inside the container checkout** — the
same working tree the agent just wrote, owned by the same user the agent runs
as. The push token (`ADW_GIT_TOKEN`) enters that exec's environment and is
consumed by a repo-local git credential helper.

A prompt-injected or otherwise hostile agent — which holds `Bash(bun:*)`, i.e.
arbitrary code — can, before the push node runs, plant a `.git/hooks/pre-push`
hook or rewrite repo-local git config (`credential.helper`, `remote.origin.url`)
inside that checkout. Git then executes those with the push token in the
environment and outbound network available; the hook can exfiltrate the token,
exit zero, and leave the run green. None of this shows in `git status`. The
host-scoping fix from adw-m4-05 stops a *redirected origin* from receiving the
token, but it does not stop a hook or repo-local config **in the checkout
itself** from reading `$ADW_GIT_TOKEN`. Separately, for a local-path origin the
container bind-mounts the origin read-write at a fixed in-container path, so the
agent's own process can rewrite or delete host-side refs and objects directly,
before gates — a plain breach of the workspace boundary (S4.2).

The operator's stated goal for container isolation (S4.5, "fully contained")
is that a possibly-hostile agent cannot reach anything outside its workspace or
lay hands on a credential. Today it can reach the push credential and the host
origin. This is the last credential/isolation boundary M4 crosses.

## Solution

The operator gets a container isolation kind where **the push credential and
the origin never touch the agent-writable checkout at all**. After gates are
green, the factory exports the gate-green commit out of the container and
performs the authenticated push from a **fresh, factory-owned host-side edge** —
a throwaway git context the agent never ran in, with hooks disabled, no
workspace-local git configuration honored, pushing the exact validated origin
URL captured at provision, with the token confined to that host-side process.
The container's checkout carries no token, no token-reading helper, and no
writable mount of the host origin. From the operator's point of view nothing
changes: `adw run --isolation container` still opens the same PR; the run's
observability artifacts are unchanged; but a hostile agent can no longer
steal the push token or corrupt the origin.

## User Stories

1. As the operator, I want the push credential to never enter the agent's
   container checkout, so that a prompt-injected agent cannot read it from the
   push environment.
2. As the operator, I want the authenticated push to run from a git context
   the agent never executed in, so that a planted `pre-push` hook never fires
   with the token.
3. As the operator, I want repo-local git configuration written by the agent
   (credential.helper, remote.origin.url, aliases) to be ignored by the push,
   so that the agent cannot redirect or hijack the credentialed operation.
4. As the operator, I want the push to target the exact origin URL captured at
   provision time, so that an agent that rewrote `origin` in the checkout
   cannot change where the token-bearing push goes.
5. As the operator, I want no read-write mount of a local-path origin into the
   container, so that the agent cannot modify or delete host-side refs/objects
   before gates run (S4.2).
6. As the operator, I want a local-path origin (used by thin-config targets and
   the test harness, N4) to still receive the attempt branch, pushed from the
   host, so that local-origin targets keep working without the rw mount.
7. As the operator, I want a URL origin (github.com and the like) to still
   receive the attempt branch exactly as today, so that real targets are
   unaffected.
8. As the operator, I want the origin URL validated before any credentialed
   push, so that a credential-bearing or malformed origin (`https://user:pat@…`,
   a quote-injection) is rejected rather than exposed on a command line or in
   config.
9. As the operator, I want the worktree isolation kind's push behavior to stay
   exactly as it is today (in-place, host keyring), so that this change is
   scoped to the container boundary and does not regress M1.
10. As the operator, I want the pipeline's node graph unchanged — the push node
    still runs once, after green gates — so that isolation kinds remain
    indistinguishable from the pipeline's point of view (S4.3).
11. As the operator, I want a failed host-side push to land the run as `blocked`
    with the reason journaled and the container/branch kept for autopsy
    (Art. VII), so that a push failure is diagnosable, not a crash.
12. As the operator, I want the bounded push-retry behavior (E6) preserved on
    the host-side edge, so that a transient network failure still retries a
    fixed number of times and no more.
13. As the operator, I want the "nothing is pushed before gates are green"
    guarantee (S2.7) to still hold at the new push boundary, so that no
    un-gated commit ever reaches origin.
14. As the operator, I want the export of the gate-green commit to carry
    exactly the committed tree the factory produced (the factory-owned commit,
    not the agent's uncommitted edits), so that what is pushed equals what
    passed gates.
15. As a security reviewer, I want a test proving a `pre-push` hook planted in
    the container checkout never executes with the push token, so that the
    core finding is regression-guarded.
16. As a security reviewer, I want a test proving the push token never appears
    in the container's process environments or filesystem at any point, so that
    the least-exposure property is machine-checked, not just asserted in prose.
17. As a security reviewer, I want a test proving a local-path origin is never
    mounted read-write into the container, so that the S4.2 breach cannot
    silently return.
18. As a maintainer, I want the push abstraction to live behind a single
    Workspace capability, so that the push node stays kind-blind and a future
    remote (E2B, M5) kind implements the same seam.

## Implementation Decisions

- **The Workspace contract owns "publish the gate-green attempt branch to
  origin", not the push node.** Today the push node runs `git status` /
  `git rev-parse` / `git push` through `workspace.exec` and layers `pushEnv()`.
  This spec replaces the in-workspace `git push` with a single Workspace push
  capability the node invokes once. This is the highest available seam: the
  push node stays pure control-flow (the S2.7 gate guard, the dirty/branch/base
  refusals, the E6 retry bound) and delegates the credentialed I/O to the kind.
  **This is a plan §5 Workspace-contract amendment — propose it, get operator
  approval, land it in the plan and the contract before implementing (the
  amendment rule; precedent: adw-m4-02 `agentSpawn`, adw-m4-03 `pushEnv`).**

- **Container kind: export-then-host-push.** After gates pass, the container
  workspace produces the gate-green commit as a git bundle (or fetches the
  attempt branch out of the container to a fresh host-side bare/temp repo the
  agent never touched). The factory then pushes from that host-side context
  with: core.hooksPath pointed at an empty/nonexistent dir (hooks disabled),
  workspace-local config not inherited (a clean `GIT_CONFIG`/env-config context,
  not the agent's `.git/config`), the origin set to the exact URL captured at
  provision (§ below), and the token supplied to that host process only. The
  token never enters the container after this change; `agentEnv()` keeps the
  Claude token (that path is unaffected), `pushEnv()` / the in-container
  credential helper are removed.

- **Origin is captured and validated at provision, pushed from the host.** The
  provision-time origin URL is the single source of truth for where the push
  goes; a checkout-time `remote.origin.url` is never trusted for the
  credentialed push. Validation rejects credential-bearing userinfo and
  shell-metacharacter/quote injection in the URL, and classifies scheme
  (https/ssh/local). This absorbs the adw-m4-06 "origin URL escaping / SSH"
  item for the push path.

- **Local-path origins: no rw mount; push from host.** The container no longer
  bind-mounts the origin read-write. A local-path origin receives the attempt
  branch from the host-side push (the host can reach the local path directly).
  This removes the S4.2 breach and the confused-deputy write surface entirely.
  This supersedes the adw-m4-06 High-4 sub-item.

- **Worktree kind: behavior preserved.** The worktree kind implements the new
  push capability as the existing in-place `git push -u origin <branch>` (host
  keyring, no token file) — byte-identical observable behavior. The pipeline
  and the worktree tests do not change.

- **SSH origins (N-of-scope note).** If the host-side edge cannot authenticate
  a scheme (e.g. ssh without the operator's key available to the push process),
  the run lands `blocked` naming the auth gap (E4 pattern), not a hang. Full
  ssh push support may remain a follow-up; the decision is that it fails
  gracefully, never silently blocks or exposes a credential.

- **Deferred findings this does NOT cover** (stay in their tickets): the hard
  container stop / crash-reclaim (adw-m4-06, review High-3) and the container CI
  repair round (adw-m4-04, review Medium-1) are unrelated remedies (abort/kill
  wiring; kind-aware reattach) and are out of this spec.

## Testing Decisions

- **What a good test is here:** it exercises the push boundary through the
  existing seams and observes *external* behavior — what reaches origin, what
  the token can and cannot touch — never internal call shapes. Real git and
  real docker are the I/O boundary (the project's standing rule; docker-gated
  suites skip loudly when docker/the image is absent but run for real in
  verify).

- **Push-node unit seam (`test/pipeline/nodes/push.test.ts`, existing).** Drive
  `makePushNode` with a fake Workspace exposing the new push capability. Assert:
  the node invokes it exactly once on green gates and never runs a tokened
  `git push` itself; the S2.7 gate guard, dirty/base/detached refusals, and the
  E6 retry bound are unchanged; a workspace whose push capability fails yields a
  graceful `blocked` naming the failure. Prior art: the existing Part A–E push
  tests and the adw-m4-03 `pushEnv` tests.

- **Container contract seam (`test/workspace/container.contract.test.ts`,
  docker-gated, existing).** Assert the three security properties directly: (1)
  a `pre-push` hook planted in the container checkout does NOT run with the
  token during publish (drop a hook that writes `$ADW_GIT_TOKEN` to a file;
  after publish, the file is absent/empty); (2) the token never appears in any
  container process env (`/proc/*/environ`) or on the container filesystem at
  any point; (3) a local-path origin is never mounted rw (inspect the
  container's mounts) yet still receives the attempt branch on the host origin
  after publish. Prior art: the adw-m4-03 Part D least-exposure/at-rest tests
  and the adw-m4-05 Part E/F image + host-scoped-helper tests.

- **Full-lane seam (`test/cli.container.test.ts`, docker-gated, existing).**
  The green-path full-lane test still opens the PR and records the attempt; add
  that the push landed on origin with the token confined host-side. Prior art:
  the existing container full-lane green/blocked tests.

- **Origin validation** is a pure function → a plain unit test (reject
  credential-bearing/quote-injection URLs; classify scheme), mirroring the pure
  `containerCredentialConfig` / `containerSpawnArgv` tests.

## Out of Scope

- Hard container stop / OS-level kill / crash-reclaim breadcrumb (adw-m4-06).
- Container CI repair-round reattach (adw-m4-04).
- Remote/E2B isolation and its push/auth (adw-m5).
- Per-run scoped git tokens (the `gh auth token` wholesale-credential
  limitation is documented in adw-m4-03; genuinely scoped tokens are a
  separate credential-minting concern).
- API-key billing (spec non-goal, N1).

## Build-log decisions (red-gate, 2026-07-18)

- **Eager provision-time git-token minting is kept** (validator Finding 2). The
  `gh auth token` thunk is called once at provision regardless of origin scheme;
  the minted token is held HOST-side by the container workspace and consumed only
  by the host-side push edge for a URL origin (a local origin needs no auth and
  the token goes unused). This preserves the full-lane `minted===1` assertion and
  least exposure (the token never nears a local push, never enters the
  container). Guarding the mint on scheme was rejected as a needless coupling.
- **Hard (non-transient) push failures burn the full E6 retry bound** rather than
  short-circuiting. The push node is kind-blind and cannot tell an ssh-auth gap
  or a validation reject from a transient network error; it retries
  `workspace.push` up to 3× and lands `blocked` with the capability's stderr in
  the reason (which names the gap). Simpler than threading a retryable flag
  through `ExecResult`, and still graceful (Story 11/ssh note).

## Live re-verification (2026-07-18) — DONE

Live container run on the scratch repo (`silouone/adw-m2-scratch`, an **https
private** origin — the URL-origin path the automated local-origin suite cannot
exercise), ticket m4s-004, PR #9 opened green. All four properties proven:

- **(a) The branch lands via the host-side edge.** `adw/m4s-004` reached the
  github origin (commit `b04f434`), PR #9 OPEN — the credentialed https push ran
  from the host, not the container.
- **(b) The token never entered the container.** On the kept container: 0 files
  under `/work /home /tmp /root` contain the token; 0 in-container process
  environments carry it (`ADW_GIT_TOKEN` or the raw value); the checkout has no
  credential helper and no `origin` remote.
- **(c) A planted pre-push hook never fires with the token** — proven live-on-
  docker by `container.contract.test.ts` (High-1 test with a positive control),
  green this session.
- **(d) The MINTED token — not the operator's keychain — authenticates.**
  Negative control reproducing the host-side edge against the https origin: the
  operator IS logged into github via the keyring (would otherwise authenticate),
  yet with the `credential.helper=` reset in place a push WITHOUT the token
  `exit 128 — Authentication failed` (keychain excluded), and WITH the minted
  token `exit 0 — [new branch]`. This is the exact regression the pure
  `hostPushArgv` order test guards.

Cleanup: container reclaimed via `adw clean --ticket m4s-004` (S4.4); scratch
target config removed from `targets/`.

## Further Notes

- This spec unifies three items from the two M4 Codex rounds that share ONE
  remedy — "push from a trusted host-side edge": the new pre-push-hook /
  repo-local-config token-exfil finding (round 2, High-1), the rw local-origin
  mount (High-4, previously in adw-m4-06), and the origin URL validation slice
  of Medium-4. On landing, adw-m4-06 should be trimmed to just the hard-stop
  (High-3) and the capture fetch-failure downgrade (Medium-2).
- Threat-model framing (carry into the ticket, do not overclaim): container
  isolation is already a net improvement over the worktree kind (where the
  agent runs on the host as the operator). This spec closes the push boundary
  so container isolation becomes safe to point at *untrusted* repositories,
  which is the whole reason the kind exists.
- The live re-verification bar (precedent adw-m4-03/05): after green, a live
  container run on the scratch repo must push via the host-side edge with the
  token proven absent from the container (fs + `/proc` scan), and a planted
  pre-push hook proven never to have seen it.
