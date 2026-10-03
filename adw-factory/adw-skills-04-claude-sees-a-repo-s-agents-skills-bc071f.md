---
id: adw-skills-04-claude-sees-a-repo-s-agents-skills-bc071f
type: feat
status: in-review
priority: 2
created: 2026-10-03
depends: [adw-skills-02-the-agent-can-actually-invoke-a-repo-skill-bc9431]
caps: {minutes: 120, turns: 600}
attempts: [{"runId":"adw-skills-04-claude-sees-a-repo-s-agents-skills-bc071f-1791045382095","branch":"adw/adw-skills-04-claude-sees-a-repo-s-agents-skills-bc071f","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-skills-04-claude-sees-a-repo-s-agents-skills-bc071f-1791045382095/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/206","provider":"claude","model":"claude-sonnet-5-5","rebased":"c6091e0ad47ba0610dc52e9a01dc31c0e2a16a2a"}]
---
# Claude sees a repo's `.agents/skills/`

> Spec: `specs/adw-v1.18-skill-routing.md`, D3, R4. It corrects v1.15 D8.

## Context

- **The bundled Claude Code CLI (2.1.287) does not discover `.agents/skills/`.**
  Probed with a fixture: only `.claude/skills/` was discovered.
- **v1.15 D8 nonetheless made `.agents/skills/` the canonical location.** A target that
  follows D8 therefore hides its skills from every Claude run.
- **The SDK `plugins: [{type: "local", path}]` option loads skills from a plugin
  directory.** Probed: a plugin skill is listed as `<plugin>:<name>`, invoked through
  `Skill`, and filtered by the explicit `skills` list.

## Requirements

- [ ] **R1: per-run local plugin.** When the worktree has `.agents/skills/*/SKILL.md`,
  `provision` writes a local plugin under `runs/<runId>/repo-skills-plugin/` exposing
  them, and the agent nodes pass it through `plugins`.
  - Whether a symlinked `skills/` directory works, or a copy is needed, is measured.
    Prefer the symlink.
- [ ] **R2: names.** The resolved list (adw-skills-02) carries the plugin-qualified
  names. A skill present under both `.claude/skills` and `.agents/skills` (D8's symlink
  layout) is offered once, under the `.claude/skills` name.
- [ ] **R3: target selection.** Target `skills: [names]` matches the bare name in
  either location.
- [ ] **R4: live probe test.** In a fixture with one skill in each location, the model
  reports being offered both, and invokes the `.agents` one.

## Verify

- `just verify` is green.
- The live probe output is pasted in the PR body.
