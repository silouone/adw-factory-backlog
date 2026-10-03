---
id: adw-skills-06-codex-sees-repo-skills-and-categories-only-d27ba2
type: feat
status: queued
priority: 2
created: 2026-10-03
depends: [adw-skills-05-global-skills-are-offered-by-category-f37d83]
caps: {minutes: 150, turns: 700}
attempts: []
---
# Codex sees repo skills and categories, nothing else

> Spec: `specs/adw-v1.18-skill-routing.md`, D5.4, D6, R6. It reverses v1.15 D6's
> "accept the leak".

## Context

- **Every Codex session catalogue has 33 entries, of which 4–5 are repo skills.** The
  rest are the operator's 24 `~/.agents/skills` (AWS) plus Codex's built-in `.system`
  skills: imagegen, openai-docs, plugin-creator, skill-creator, skill-installer.
- **Probed on 0.155.1 with `codex debug prompt-input`:**
  - `-c 'skills.config=[{name="aws-iam",enabled=false}]'` removes exactly that skill
    (34 → 33);
  - `-c skills.bundled.enabled=false` removes the 5 built-in skills;
  - a **path** entry naming a directory does nothing.

## Requirements

- [ ] **R1: pure argv extension.** `codexSkillArgs(hostSkillNames)` returns the
  `-c skills.bundled.enabled=false` flag plus one `skills.config` disable entry per
  host skill name. It is appended by `codexExecArgv`, and has table tests, including
  names that need TOML quoting.
- [ ] **R2: host discovery at the edge.** Names come from `~/.agents/skills`,
  `$CODEX_HOME/skills` (not `.system`) and any `.agents/skills` in an ancestor of the
  worktree. Repo skills (the worktree's own `.agents/skills`) are never disabled.
- [ ] **R3: category skills.** The category skills from adw-skills-05 are written to
  the worktree's `.agents/skills/adw-<cat>/` and hidden by a per-worktree exclude file
  (the `cmc.setup.sh` idiom, with `extensions.worktreeConfig`).
  - The worktree must still be clean after provision.
  - A name collision with a repo skill refuses the run, naming it.
- [ ] **R5: name collisions.** A `name` selector cannot tell sources apart, so
  disabling a host skill would also disable a repo skill with the same name.
  - First measure whether a `path` selector naming the exact `SKILL.md` **file** works;
    only a directory path has been shown to fail. If it works, use path selectors.
  - Otherwise, a host name that collides with a repo skill is **not** disabled. The
    collision is journaled in `run-start`, and the repo skill wins.
- [ ] **R4: provenance.** `run-start` lists the disabled host names, so the
  suppression is visible.

## Verify

- `just verify` is green.
- **Live, required:** `codex debug prompt-input` in a provisioned cmc worktree lists
  exactly the 4 repo skills plus the opted-in categories.
- One `just run-codex` on a cqc ticket. Its session catalogue matches, the before/after
  catalogue token counts go in the PR body, and `skill-use` journals at least one read.
- If `skills.config` turns out not to hold on `codex exec`, stop and report: do not ship
  a partial suppression.
- **Merge gate:** this ticket does not merge unless the PR body carries the live
  `prompt-input` catalogue and a `just run-codex` runId.
