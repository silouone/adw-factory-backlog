---
id: adw-m4-08-ssh-push-support
type: feat
status: blocked
priority: 1
created: 2026-07-18
epic: adw-m4
depends: [adw-m4-07-host-side-push]
attempts: [{"runId":"adw-m4-08-ssh-push-support-1789114727413","branch":"adw/adw-m4-08-ssh-push-support","workspace":"/Users/silouane/adw-factory/runs/adw-m4-08-ssh-push-support-1789114727413/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# SSH origins through the host-side push edge

> The adw-m4-07 residue, ticketed so it doesn't evaporate (operator-endorsed
> 2026-07-18). m4-07's SSH note deliberately scoped ssh to "fails gracefully,
> never silently blocks or exposes a credential": the host-side edge pushes an
> ssh origin with whatever the host's ssh setup provides, and an auth gap
> lands `blocked` naming the failure. That graceful failure is the CURRENT
> state, not the end state.

## Context (refine at pickup)

`hostSidePush` (src/workspace/container.ts) classifies ssh origins via
`validateOriginUrl` and pushes them with NO credential flags — the fresh
edge repo's `git push` falls through to the host's ambient ssh
configuration (agent, default keys, ~/.ssh/config). Known gaps:

- **Host-key verification in the fresh edge context:** a first-contact ssh
  origin prompts/fails on known_hosts in a non-interactive push process;
  nothing pins the behavior (accept-new? fail loudly naming the remedy?).
- **Key scoping:** the operator's ambient ssh identity is broader-scoped
  than the minted https token the edge was built around — decide whether
  that asymmetry is accepted (host-side edge is factory-owned, agent can't
  reach it) or whether a per-target `core.sshCommand`/identity pin is
  wanted.
- **No live proof:** no scratch run has ever pushed an ssh origin through
  the edge; the graceful-failure path itself is only unit-pinned.

## Requirements (refine at pickup)

- [ ] Decide + document the ssh trust/identity policy for the edge
      (operator decision), then implement: deterministic known_hosts
      behavior and (if chosen) identity pinning via the pure `hostPushArgv`
      — order-tested like the https helper-reset.
- [ ] Failure messages name the gap and remedy (Art. IX) — distinguishable
      from a token-auth failure.
- [ ] Live verify: one scratch push over an ssh origin (or a deliberate
      auth-gap run proving the named graceful failure), per the m4-07
      negative-control lesson.

## Out of scope

Remote/E2B (adw-m5); any in-container ssh (the checkout stays push-free).
