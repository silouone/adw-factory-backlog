---
id: adw-route-09-codex-rounds-take-a-gear-and-rounds-are-preflighted-6758ca
type: feat
status: queued
priority: 2
created: 2026-10-03
depends: [adw-route-02-every-agent-node-is-routable-5d55f8]
---
# A codex CI or rebase round takes a gear, and every round is preflighted

> Spec: `specs/adw-v1.16-model-routing.md`. Follow-up to adw-route-02 (PR #200),
> which was merged by hand onto adw-route-03.

## Context

- `resolveRoundProfile` (src/pipeline/lanes/shared.ts) still returns early for a
  codex attempt, with `attempt.model ?? CODEX_DEFAULT_MODEL` at effort high. On a
  codex attempt, `agents.ci-repair` and `agents.rebase-resolve` are ignored,
  even though route-03 gave every other stage the codex column of the ladder.
- The model ↔ CLI preflight (route-02 R4) checks only the run's own stage map. A
  CI or rebase round whose resolved profile is a bounded model is never checked.

## Requirements

- [ ] **R1:** on a codex attempt, `resolveRoundProfile` resolves through the
      same chain on the codex column (`profileOrThrow(..., "codex")`, floor
      `FACTORY_DEFAULT_CODEX_PROFILE`). The attempt's recorded model stays the
      whole-ticket level, and with nothing configured the argv is unchanged.
- [ ] **R2:** a round runs `modelCliRefusal` on its resolved profile before it
      spawns the agent. A refusal ends the round with that reason and charges
      no session.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
