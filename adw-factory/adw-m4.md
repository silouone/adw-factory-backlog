---
id: adw-m4
type: epic
status: done
priority: 3
created: 2026-07-14
depends: [adw-m3, adw-m6]
children:
  - adw-m4-01-container-image
  - adw-m4-02-container-workspace
  - adw-m4-03-container-auth
  - adw-m4-04-ci-round-container
  - adw-m4-05-review-fixes
  - adw-m4-06-hard-stop-container
  - adw-m4-07-host-side-push
  - adw-m4-08-ssh-push-support
attempts: []
---
# EPIC M4 — Container workspace

Second `Workspace` kind: fully contained local execution (OrbStack/Docker)
behind the exact interface proven in M1 — the pipeline must not change
(S4.3, S4.5). **Post-v1 expansion** (plan §8 as reordered 2026-07-14: M6
shakedown precedes it — now binding via `depends: [adw-m3, adw-m6]`).

**Children refined 2026-07-15 at pickup** (plan §8, protocol §3) — each
carries full requirements + build protocol and awaits operator approval
before `in-progress`.

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m4-01-container-image | committed Dockerfile: bun+git+claude, zero secrets | S4.5, N1 |
| adw-m4-02-container-workspace | `container.ts` + agent-in-container seam + reused contract suite | S4.3 S4.5 |
| adw-m4-03-container-auth | Claude token AND git-push token, env-only, live full-lane run | N1 |

## Refinement findings (2026-07-15)

- **The under-analyzed core is the agent-execution seam** (m4-02): with
  worktrees the agent runs on the host; "fully contained" means claude
  runs inside the container, driven by the SDK over stdio through a
  docker-exec shim or a per-kind query seam. m4-02 opens with a mandatory
  spike; if the `Workspace` contract needs extending, the amendment rule
  applies — stop and propose, never silently diverge.
- **Auth is two credentials, not one** (m4-03): the subscription token for
  the in-container agent AND a git-push token for the push node's
  in-container `git push`. Both env-only, never at rest.
- The live full-lane container run lands in m4-03 (it needs auth); m4-02
  proves the lane with the fake query + the unchanged contract suite.

## Exit criteria

A toy chore completes the full lane under `--isolation container` with
identical pipeline behavior and complete observability artifacts (agent
spans still parented under the run — S5.3). This is m4-03's live verify.
