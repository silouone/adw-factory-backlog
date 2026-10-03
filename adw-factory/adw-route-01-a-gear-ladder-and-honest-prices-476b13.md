---
id: adw-route-01-a-gear-ladder-and-honest-prices-476b13
type: feat
status: done
priority: 1
created: 2026-10-03
depends: []
caps: {minutes: 150, turns: 700}
attempts: [{"runId":"adw-route-01-a-gear-ladder-and-honest-prices-476b13-1791018926955","branch":"adw/adw-route-01-a-gear-ladder-and-honest-prices-476b13","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-route-01-a-gear-ladder-and-honest-prices-476b13-1791018926955/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/189","provider":"claude","model":"claude-sonnet-5-5"}]
---
# One gear ladder for both providers, priced at today's rates

> Spec: `specs/adw-v1.16-model-routing.md`: the gear ladder, ¹ note, and D1–D3, D11.
> First ticket of the model-routing track. Everything else stacks on it.

## Context

- **The profile registry is never used.** It has four names (`deep`, `standard`, `swift`, `cheap`). Across 1,538 agent node-ends only `standard` ever ran.
- **`cheap` breaks two rules:**
  - it sends the bare alias `haiku`, which v1.8 D1 forbids;
  - it asks for `effort: low`, below the floor the operator set (D1).
- **The rate card is stale.**
  - It prices Sonnet at 3/15/0.30 and Opus at 15/75/1.50. Published prices on 2026-10-03 are Sonnet 5.5 at 2/10/0.20/2.50 and Opus 5.5 at 4/20/0.20/5.
  - Fable 5.1 (10/50/0.25/12.50), gpt-6-luna, gpt-reserve and gpt-5.6-terra are missing, so runs on them show no spend estimate.
  - Every Claude spend figure is overstated by about 1.5×.
- **Haiku 4.5 does not support `effort`.** It takes a fixed thinking budget (`thinking: {type: "enabled", budgetTokens}`). Probed 2026-10-03: `--effort` is accepted but doesn't steer Haiku's thinking.

## Requirements

- [ ] **R1: the ladder.** Gears G0, G0+, G1, G2, G3, G4, G5. Each carries a Claude `{model, reasoning}` and a Codex `{model, effort}` exactly as in the spec's table, with pinned exact slugs (`claude-haiku-4-5-20251001`, `claude-sonnet-5-5`, `claude-opus-5-5`, `claude-fable-5-1`, `gpt-6-luna`, `gpt-5.6-terra`, `gpt-6-sol`, `gpt-6-astra`).
- [ ] **R2: reasoning per model kind is a typed union.** Adaptive-thinking models (Sonnet 5.5, Opus 5.5, Fable 5.1) get `effort`; Haiku gets a thinking budget (named constants: *medium* 8k, *high* 24k). Haiku with an `effort` cannot be represented. The Claude agent-query options carry whichever one the gear has, never both.
- [ ] **R3: legacy names keep working.** `standard`→G2, `swift`→G1, `deep`→G3. `cheap` is removed: a ticket or target naming it is refused at parse or load time with a message pointing to G0.
- [ ] **R4: gear ids are accepted everywhere a profile name is** (`agent:`, `agents:`, target `agents`). An unknown value is refused, naming the ticket or target and the value.
- [ ] **R5: the factory default stays G2.** With no ticket or target setting, every node's resolved model and reasoning is byte-identical to today (Sonnet 5.5, effort high). This is the zero-behaviour-change guarantee.
- [ ] **R6: rate card.** Today's published prices for Sonnet 5.5, Opus 5.5, Haiku 4.5 and Fable 5.1, plus exact-slug keys for gpt-6-luna, gpt-reserve and gpt-5.6-terra, read from OpenAI's model pages, with `asOf` dates. The rate-card test asserts **every gear model has a rate**, so a future gear without one fails the suite.
- [ ] **R7: journal.** Agent node-end usage carries `gear` beside the existing `model`/`effort`/`profile`, and Haiku nodes carry their thinking budget instead of an effort.

## Out of scope

Routing logic, escalation, Codex per-stage effort (route-03), ci-repair/rebase-resolve (route-02).

## Post-merge check (operator)

One chore dispatched with `agent: G0` shows `claude-haiku-4-5-20251001` and a thinking budget in the journal; one with `agent: G5` shows `claude-fable-5-1`.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
