---
id: adw-skills-02-the-agent-can-actually-invoke-a-repo-skill-bc9431
type: feat
status: queued
priority: 1
created: 2026-10-03
depends: []
caps: {minutes: 150, turns: 700}
attempts: []
---
# The agent can actually invoke a repo skill

> Spec: `specs/adw-v1.18-skill-routing.md`, §1, D1, D2, R1, R2. It reverses v1.15 D1 and D5.

## Context

- **`skills:"all"` is inert.** All 63 runs that requested it journal
  `tools = [Bash, Edit, Glob, Grep, Read, Write]` and made zero `Skill` calls. Asked
  directly, the model reports being offered **no** skills.
- **The cause is the `tools` whitelist** (`AGENT_TOOLS`, `build.ts:360`). `tools` sets
  the base set of built-in tools, and `Skill` is one of them. The SDK note about not
  adding `Skill` refers to `allowedTools` only.
- **The fix is probed.** Adding `"Skill"` to `tools` makes the 5 sabado skills visible
  and invocable.
- **Reviewers inherit the gap.** `REVIEW_TOOLS` derives from `AGENT_TOOLS`, so they are
  inert too.
- **`"all"` leaks non-repo skills** (`plugin-authoring`, `design`, `doctor`). An
  explicit name list closes the leak (probed).

## Requirements

- [ ] **R1: target `skills` semantics** (`src/targets/loader.ts`).
  - Absent → every repo skill, a list → those, `[]` → off.
  - `"all"` is refused at load, naming the target and pointing to absent or a list.
- [ ] **R2: pure resolution.** `resolveRepoSkills(dirListing, declared)` returns the
  explicit name list.
  - It scans the worktree's `.claude/skills/*/SKILL.md` after `provision`.
  - A declared name missing from the worktree is refused before any agent node,
    naming the ticketId, the target and the name.
  - The CLI resolves once per run and threads the list where `resolvedSkills` goes today.
- [ ] **R3: the SDK always receives the explicit list**, never the string `"all"`.
- [ ] **R4: `Skill` in the tool surface iff the list is non-empty.**
  - This covers every agent node: build, plan, test, repair, ci-repair,
    rebase-resolve and review-fix.
  - It also covers the review read-only set.
  - An empty list leaves the surface byte-identical to today, with a test asserting it.
- [ ] **R5: live probe test** (skipped without credentials, like the other live suites).
  - Build `baseOptions` the way `makeBuildNode` does, in a fixture repo holding one
    `.claude/skills/x/SKILL.md`.
  - Assert `Skill ∈ init.tools`.
  - Ask the model, with no tools, to list the skills it was offered, and assert it
    answers exactly `x`.
- [ ] **R6: journal.** `node-start.skills` carries the resolved list, not `"all"`.
- [ ] **R7: config.** `targets/adw-factory.json` drops `"skills": "all"` (absent now
  means on). The README's Targets paragraph is updated to match.

## Verify

- `just verify` is green.
- One live `just run` on a sabado ticket, with the runId pasted in the PR body:
  - `node-end.agentConfig.tools` includes `Skill`;
  - the `cache_creation` delta on the first call is recorded (≈ +1k expected).

- **Merge gate:** this ticket does not merge unless the PR body carries a live runId
  whose `init.tools` contains `Skill`. A probe test that skips without credentials is
  not evidence: adw-skills-01 went `done` that way and shipped inert.

## Conflict note

This ticket edits `AGENT_TOOLS`, the review tool set and every node's `baseOptions`.
adw-route-01 (#189, in review) and adw-route-02 (queued) change the same code. Rebase
onto whichever lands first, and expect conflicts.

## Out of scope

`.agents/skills/` on Claude (adw-skills-04), global categories (05/06), journaling of
skill use (03).
