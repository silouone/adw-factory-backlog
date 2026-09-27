---
id: adw-profile-02-a-provider-per-stage
type: feat
status: blocked
priority: 2
created: 2026-09-17
depends: [adw-profile-01-a-model-and-effort-per-stage, adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run]
attempts: [{"runId":"adw-profile-02-a-provider-per-stage-1789738890261","branch":"adw/adw-profile-02-a-provider-per-stage","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-profile-02-a-provider-per-stage-1789738890261/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Let the reviewer be a different model family than the builder

> `specs/adw-v1.8-agent-profiles.md` R5, R6, R7, and decision D2. The second of
> two: `adw-profile-01` built the profile, its registry and its precedence;
> this ticket lets a profile's **provider** actually take effect per stage.

## Context

The provider is resolved ONCE per run and never varies:

- `src/cli.ts:288` — `flags.provider ?? target.provider ?? "claude"`, run-scoped.
- `src/cli.ts:769` — `const selectedQuery: AgentQuery = provider === "codex" ?
  requireCodexQuery(deps) : deps.query`. One binding; every node closes over it.

So a profile whose provider is `codex` cannot take effect: the lane has no way
to reach a second `AgentQuery`. This ticket replaces the single binding with a
**router** — `(profile) => AgentQuery` — so a node picks its binding from its
own resolved profile.

**Why this matters more than "docs on Codex."** The reviewer M9 just landed
(`review-standards`/`review-spec`) reads the same `agentConfig` the build
stages read, so it judges work produced by its own model class and shares its
blind spots. Cross-provider review is the single change most likely to move
M9's own §6 exit metric — operator review minutes per PR. That metric is the
instrument that decides whether this ticket paid for itself.

## Why it depends on `adw-bug-11`

`CODEX_DEFAULT_MODEL = "gpt-5.4"` is rejected on a ChatGPT-account login, so
every Codex run currently dies at its first agent node. Routing a stage to
Codex before that is fixed produces an unrunnable feature.

## Deliverables

The query router in `src/cli.ts` and the lane deps; the per-stage attempt
record in `src/intake/ticket.ts` + `src/pipeline/nodes/open-pr.ts` +
`src/pipeline/nodes/ci-round.ts`; the dispatch-time isolation refusal; and
Codex-path effort/model threading in `src/codex-query.ts`.

## Requirements

- [ ] **R5 — the query router.** A stage's resolved profile selects its
      `AgentQuery`. `selectedQuery` stops being run-scoped. The
      `onAdapterEvent` journal wrapper (`cli.ts:790`) must still wrap EVERY
      binding the router can return — a synthesized-exit teardown on a
      Codex-routed stage must still reach the journal.
- [ ] **Codex-path effort and model.** `codex-query.ts` builds argv internally
      and never reads `AgentNodeConfig`: `CODEX_REASONING_EFFORT` is baked in at
      lines 468 and 483. A stage's profile effort/model must thread through the
      argv builder. The determinism guarantee stays: a run NEVER inherits the
      operator's `~/.codex/config.toml` — `-c model_reasoning_effort=` is still
      always passed, never omitted.
- [ ] **R6 — the attempt record becomes a per-stage map** (D2): what each stage
      actually ran on. Readers MUST accept both shapes — every attempt already
      on disk carries scalar `provider`/`model`, and a pre-provenance attempt
      carries neither (`ticket.ts:43-52` documents both compat cases). A
      migration of existing tickets is NOT in scope; the reader adapts.
- [ ] **CI-repair resume** reads the per-stage map and resumes the repaired
      stage on the provider and model THAT stage used — not the run's, which no
      longer exists as a single value. `ci-round.ts` currently routes by "the
      ORIGINATING attempt's provider"; that contract is preserved, now
      per-stage.
- [ ] **R7 — isolation refusal at dispatch.** `provider "codex" only supports
      worktree isolation` (`cli.ts:296`). A profile routing ANY stage to Codex
      on a container/remote run must refuse BEFORE provisioning — naming the
      stage, its profile, and the isolation kind. Never mid-run, after tokens
      are spent.
- [ ] **Codex auth is checked once, at dispatch**, if any stage's profile
      resolves to Codex — the existing `cli.ts:416` pre-flight, now conditional
      on the resolved profile set rather than on a run-scoped flag. A run that
      would fail at stage 7 for missing auth must refuse at stage 0.
- [ ] **R8 — journaled.** The resolved provider rides in the same typed details
      bag `adw-profile-01` established. A mixed-provider run must be readable as
      such from artifacts alone.

## Verify

- [ ] Red tests first. At minimum:
      a run where `build` routes to one provider and `review-spec` to another,
      asserted on WHICH fake query each node was handed;
      the per-stage attempt map round-trips through write → read;
      a LEGACY scalar attempt still resumes correctly (compat);
      a pre-provenance attempt (neither field) still resumes correctly;
      a Codex-routed stage on container isolation refuses at dispatch, with no
      workspace provisioned (asserted on the provision spy, not just the exit
      code);
      missing Codex auth refuses at dispatch when any stage resolves to Codex;
      Codex argv carries the profile's effort, never the module constant.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] Operator-executed (real spend, not agent-executed): one live mixed run —
      Claude builder, Codex reviewer — reaching `open-pr`, with both providers
      visible in the journal and the attempt record.

## Out of scope

Codex container/remote parity (still worktree-only, unchanged); `ci-round`'s
own `CI_MODEL` (spec §9 open question — resuming a session on a different model
than it was created with is a provider-semantics question, not a routing one);
automatic provider selection by ticket difficulty; migrating existing tickets'
scalar attempt records.
