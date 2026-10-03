---
id: adw-skills-05-global-skills-are-offered-by-category-f37d83
type: feat
status: queued
priority: 2
created: 2026-10-03
depends: [adw-skills-03-every-skill-the-agent-loads-is-journaled-bf8d35]
caps: {minutes: 180, turns: 800}
attempts: []
---
# Global skills are offered by category (Claude path)

> Spec: `specs/adw-v1.18-skill-routing.md`, §2, D4, D5.1–3, D9, R5. The taxonomy is
> decided: §9 classifies all 76 global skills into 7 categories.

## Context

- **Global skills are invisible today.** The operator's global skills live in
  `~/.claude/skills` (73) and `~/.agents/skills` (24, all AWS), and no factory run sees
  them on purpose.
- **Exposing them flat would swamp the repo skills.** On Codex today, the 24 AWS skills
  cost ≈ 4,558 tokens next to 311 for the repo's own.
- **The design is one catalogue entry per category, sub-skills read on demand.** Probed
  end to end: a plugin skill `adw:aws` is invoked, and the model then `Read`s the
  sub-skill and follows it. The `Read` outside cwd is **refused** headless unless the
  directory is in `additionalDirectories`.

## Requirements

- [ ] **R1: `skills/categories.json`** in this repo.
  - Loader and validation: unknown keys are refused; a category matching nothing is
    refused, naming it.
  - Each category has an optional `factory: boolean`, default `true`.
  - Content is exactly spec §9's table, and each description names its domain's nouns.
  - A test asserts that every name in §9 classifies into exactly one category.
  - `just skills-unclassified` prints the global skills that match no category.
- [ ] **R2: pure global discovery.** `discoverGlobalSkills(listings)` covers
  `~/.claude/skills` then `~/.agents/skills`: the first source wins, and collisions are
  returned for the journal.
  - `classify(catalogue, categories)` returns `{byCategory, unclassified}`.
- [ ] **R3: target `globalSkills: string[]`.**
  - Absent → every category with `factory` ≠ `false` (operator decision, 2026-10-03);
    `[]` → none.
  - An unknown category, or one with `factory: false`, is refused at load, naming the
    target.
- [ ] **R4: snapshot.** The included sub-skill trees are copied into
  `~/adw/skills-cache/<sha256>/`, made read-only, and reused while the hash matches.
  - Pure hashing, with I/O at the edge.
  - `run-start` carries the hash, the categories and `unclassified`.
- [ ] **R5: generated category skill.** For each category the target opted into, a
  pure render produces its `SKILL.md`: the category description, then one line per
  sub-skill with its name, description and absolute snapshot path.
  - Golden-file test.
- [ ] **R6: Claude wiring.**
  - A per-run plugin `adw` under `runs/<runId>/skills-plugin/`.
  - `adw:<cat>` is appended to the resolved list.
  - `additionalDirectories: [snapshotDir]` on every agent node, review included.
- [ ] **R7: isolation (D9).** Container and remote runs drop global categories with a
  journaled line. They never refuse.

## Verify

- `just verify` is green.
- A live probe test: a fixture category with a secret word in a sub-skill. The model
  invokes `adw:<cat>`, `Read`s the sub-skill and returns the word, and `skill-use`
  journals both steps.
- A live adw-factory run with the default (all factory categories), with its
  `run-start` and the catalogue token delta in the PR body.
- **Merge gate:** this ticket does not merge unless that live runId is in the PR body.
