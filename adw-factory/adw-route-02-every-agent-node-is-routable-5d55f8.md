---
id: adw-route-02-every-agent-node-is-routable-5d55f8
type: feat
status: in-review
priority: 1
created: 2026-10-03
depends: [adw-route-01-a-gear-ladder-and-honest-prices-476b13]
attempts: [{"runId":"adw-route-02-every-agent-node-is-routable-5d55f8-1791027763132","branch":"adw/adw-route-02-every-agent-node-is-routable-5d55f8","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-route-02-every-agent-node-is-routable-5d55f8-1791027763132/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"adw-route-02-every-agent-node-is-routable-5d55f8-1791027763132","branch":"adw/adw-route-02-every-agent-node-is-routable-5d55f8","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-route-02-every-agent-node-is-routable-5d55f8-1791027763132/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/200","provider":"claude","model":"claude-sonnet-5-5","note":"hand salvage after review-standards unknown-field verdict; merged onto route-03"}]
---
# No agent node bypasses the gear system, and dispatch refuses a gear the CLI can't run

> Spec: `specs/adw-v1.16-model-routing.md`, answering v1.8 §9 and v1.8 D2/R6.

## Context

- **Two agent nodes are outside the profile chain.** `ci-repair` (CI round) and `rebase-resolve` (rebase round) both read a hardcoded CI model constant or the attempt's scalar model, with effort hardcoded to `high`. Neither is in the agent stage-name list, so no ticket, target or router can reach them.
- **The attempt record is a scalar** `provider`/`model`. Once nodes run on different gears it can't say what ran, or where a blocked run died. Cross-run escalation (route-06) needs both.
- **An unrunnable model is discovered mid-run.** Opus 5.5 needed CLI ≥ 2.1.280 and failed with an API 400 after provisioning (adw-model-01).

## Requirements

- [ ] **R1:** `ci-repair` and `rebase-resolve` are agent stage names. Both resolve a gear through the same chain as every other stage. The hardcoded CI model constant is removed. With nothing configured they resolve to G2, which matches today's behaviour.
- [ ] **R2:** the attempt record carries a per-stage map `{stage → {gear, provider, model}}`, plus `blockedAt: {node, reason}` on a blocked attempt. Readers accept the legacy scalar shape and a pre-provenance attempt with neither (v1.8 D2).
- [ ] **R3:** a CI or rebase round on an attempt written before this ticket still resolves (legacy scalar → G2-equivalent on that model).
- [ ] **R4: preflight.** Before provisioning, every resolved gear's model is checked against a table of minimum CLI versions (Opus 5.5 ≥ 2.1.280; Fable 5.1 ≤ 2.1.287, probed 2026-10-03) and against the installed agent binary for the run's isolation kind. A miss refuses with exit 2, naming the ticket, stage, model and both versions.
- [ ] **R5:** the refusal happens before any workspace or token is spent (same seam as the codex+container refusal).

## Out of scope

Choosing gears (route-05), Codex per-stage effort (route-03).

## Verify

- [ ] Red tests first (Art. I), confirmed failing on `main`, then green.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test`, all green.
