---
id: adw-skills-03-every-skill-the-agent-loads-is-journaled-bf8d35
type: feat
status: in-progress
priority: 1
created: 2026-10-03
depends: [adw-skills-02-the-agent-can-actually-invoke-a-repo-skill-bc9431]
caps: {minutes: 120, turns: 600}
attempts: [{"runId":"adw-skills-03-every-skill-the-agent-loads-is-journaled-bf8d35-1791045377642","branch":"adw/adw-skills-03-every-skill-the-agent-loads-is-journaled-bf8d35","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-skills-03-every-skill-the-agent-loads-is-journaled-bf8d35-1791045377642/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/207","provider":"claude","model":"claude-sonnet-5-5","rebased":"c6091e0ad47ba0610dc52e9a01dc31c0e2a16a2a"}]
---
# Every skill the agent loads is journaled

> Spec: `specs/adw-v1.18-skill-routing.md`, D8, R3.

## Context

- **Nothing records which skill an agent actually used.** We can't tell whether a skill
  description triggers, or whether a category earns its place.
- **This lands before the global categories** (05/06), so that repo-skill use has a
  baseline to compare against.

## Requirements

- [ ] **R1: event `skill-use {node, skill, via}`**, where `via` is `"skill-tool" | "read"`.
  It is emitted from the agent stream the node already consumes; no new I/O.
- [ ] **R2: Claude.**
  - A `Skill` tool_use emits `via: "skill-tool"`, with `skill` taken from its input.
  - A `Read` tool_use whose path ends in `/SKILL.md` emits `via: "read"`. The `skill`
    name is the parent directory name.
- [ ] **R3: Codex.** A command whose argv names a `…/SKILL.md` path emits `via: "read"`.
- [ ] **R4: pure classifier.** `classifySkillUse(toolUse)` → event | null, with table
  tests for both providers, including paths that are *not* skills (for example
  `docs/SKILL.md.bak`).
- [ ] **R5: `run-start`** carries the resolved repo-skill list (spec D8).
- [ ] **R6: `just skills <runId>`** prints node, skill and via, one line each, read from
  `.event` like every other recipe.

## Verify

- `just verify` is green.
- Replay a banked cqc Codex journal and a Claude transcript through the classifier
  fixture: the cqc-fe-39 session's reads of `cqc-frontend`, `gen-from-openapi` and
  `plan-cmc-ui` are classified.
