---
id: adw-m1
type: epic
status: done
priority: 1
created: 2026-07-14
depends: [adw-m0]
children:
  - adw-m1-01-ticket-contract
  - adw-m1-02-selection
  - adw-m1-03-status-writer
  - adw-m1-04-journal
  - adw-m1-05-engine
  - adw-m1-06-worktree-workspace
  - adw-m1-07-target-config-prompt
  - adw-m1-08-gates-node
  - adw-m1-09-agent-nodes
  - adw-m1-10-lane-cli
  - adw-m1-11-integration
attempts: []
---
# EPIC M1 — Loop proven

The core loop: a valid chore ticket travels validate → dispatch →
provision(worktree) → assemble-prompt → build(agent) → gates, with a
≤3-round same-session repair loop, fully journaled. Ends at a green **local**
branch — push/PR are M2, traces are M3.

## Children

| Ticket | Scope | Key criteria |
|--------|-------|--------------|
| adw-m1-01-ticket-contract | frontmatter parse + validation | S1.2 S1.3 |
| adw-m1-02-selection | named / priority / oldest selection | S1.1 |
| adw-m1-03-status-writer | committed transitions = dispatch lock | N3 E1 E7 |
| adw-m1-04-journal | JSONL run journal | S5.1, Art. VI |
| adw-m1-05-engine | blueprint state machine, rounds, ceilings, abort | S2.3 S2.5 S2.6, Art. V |
| adw-m1-06-worktree-workspace | Workspace contract + worktree impl | S4.2–S4.5 E2 E5 N6 |
| adw-m1-07-target-config-prompt | target config loader + assemble-prompt | S1.4 N4 N5 |
| adw-m1-08-gates-node | lint→typecheck→test as code | S2.1 E3 |
| adw-m1-09-agent-nodes | build + repair over the Agent SDK | S1.4 S2.2 S6.3 E4 |
| adw-m1-10-lane-cli | chore lane config + `adw run` | S4.1 S6.1 S6.3 |
| adw-m1-11-integration | live end-to-end exit gate | plan §8 M1 exit |

## Exit criteria

A toy chore ticket in a fixture repo reaches a green local branch with a
complete journal; a forced-failure variant exercises the repair loop live.
