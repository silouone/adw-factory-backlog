---
id: adw-m7
type: epic
status: done
priority: 3
created: 2026-07-19
depends: [adw-m1]
children:
  - adw-m7-01-codex-worktree
  - adw-m7-02-codex-capture
  - adw-m7-03-codex-ci-auth-provenance
attempts: []
---
# EPIC M7 — Multi-provider (Codex)

A second agent **provider** behind the same injected `AgentQuery` seam, so a
target (or a single run) can execute on **Codex** end-to-end instead of Claude.
Provider is a dimension **orthogonal to isolation kind** (M4/M5): it lands on
the **worktree** kind first; container/e2b Codex parity is Tier-2, deferred.

**Post-v1 expansion.** Direction agreed 2026-07-19 (memory: "Codex
multi-provider direction"; full analysis in
`ai_docs/2026-07-19-codex-multi-provider-report.md` §1). Motivating scenario:
subscription tokens run out → flip config → the factory runs on Codex.

## Why this is not M5

M5 is the E2B/**isolation** epic (a third `Workspace` kind). Codex is a
**provider** swap at the SDK boundary (`live-query.ts` → a sibling
`codex-query.ts`), entirely orthogonal to which sandbox the agent runs in. The
pipeline is already agent-blind — build/repair consume the injected `AgentQuery`
seam and only `live-query.ts` touches the Claude SDK.

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m7-01-codex-worktree | Provider-shaped `AgentQuery` boundary + `codex-query.ts` (`codex exec --json` + resume) + CLI/target provider selection + per-provider default model + env-policy | S6.3, N1 |
| adw-m7-02-codex-capture | Hooks-based cLens capture for Codex runs (`makeCodexClensHooks`) — coarse, refine at pickup | S5.3 |

## Exit criteria

A chore completes the full worktree lane under `--provider codex` (dispatch →
build → gates → commit → push → PR), including one in-session repair round on a
CI-red PR (`codex exec resume`); the Claude path stays byte-identical when no
provider is selected.
