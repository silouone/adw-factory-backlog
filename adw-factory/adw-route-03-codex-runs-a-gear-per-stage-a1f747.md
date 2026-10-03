---
id: adw-route-03-codex-runs-a-gear-per-stage-a1f747
type: feat
status: in-progress
priority: 1
created: 2026-10-03
depends: [adw-route-01-a-gear-ladder-and-honest-prices-476b13]
attempts: [{"runId":"adw-route-03-codex-runs-a-gear-per-stage-a1f747-1791033542847","branch":"adw/adw-route-03-codex-runs-a-gear-per-stage-a1f747","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-route-03-codex-runs-a-gear-per-stage-a1f747-1791033542847/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/196","provider":"claude","model":"claude-sonnet-5-5","rebased":"110a720d5eeede9315aea86cb73ec2911c445380"}]
---
# Codex runs a gear per stage, not one model at one effort

> Spec: `specs/adw-v1.16-model-routing.md`, the Codex column of the ladder.

## Context

- **9 of 14 targets are Codex**, and each gets one placeholder profile for every stage.
- **The Codex binding sets reasoning effort to `high` unconditionally** and never reads the effort in the agent-query options. Per-stage routing can't even be configured there.
- **The catalog offers more models:** gpt-6-luna and gpt-5.6-terra (fast, balanced), gpt-6-sol and gpt-6-astra, with effort from `low` to `ultra`.

## Requirements

- [ ] **R1:** on a Codex run, each agent node resolves its gear through the same chain as Claude (ticket per stage → ticket whole → target lane×stage → target lane → default G2) and runs the gear's Codex `{model, effort}`.
- [ ] **R2:** the Codex binding passes the per-node effort and model in its argv. The hardcoded effort constant is removed, and the factory still never inherits the operator's global Codex config (the adw-perf-02 rule).
- [ ] **R3:** a whole-ticket `model:` override (for example the 87 `gpt-5.6-sol` tickets) still applies to every stage at today's effort. Zero behaviour change for pinned tickets (D8).
- [ ] **R4:** with nothing configured, a Codex run's argv is identical to today (gpt-6-sol, effort high).
- [ ] **R5:** a gear valid only on Claude (none today) or an effort outside the Codex vocabulary is refused at parse or load time.

## Out of scope

Per-stage **provider** (adw-profile-02). Container/remote Codex.

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
