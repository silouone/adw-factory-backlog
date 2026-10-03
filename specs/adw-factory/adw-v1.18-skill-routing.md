# Amendment v1.18: skills actually reach the agent, and global skills are offered by category

Status: **approved** by the operator, 2026-10-03 (direction, the §9 taxonomy and the defaults, in-session).
Source note: `ai_docs/2026-10-03-skills-consumption-audit.md`. That note is local and
gitignored, so every number it supports is repeated here with its probe.

Amends `adw-v1.15-skills.md`: it reverses **D1** (absent → `[]`), **D5** (the SDK adds
`Skill` by itself) and **D8**'s claim that Claude Code reads `.agents/skills/`. It also
closes the Codex host leak that **D6** accepted. v1.15 D2, D3, D4 and D7 stand.

## 1. Trigger: v1.15 shipped inert

v1.15 R6 was the live probe that should have proved this feature works. PR #130 deferred
it. D5 itself said that if the probe failed, "the feature is inert and the implementation
must stop and report". It is inert. The evidence below is that R6 result, taken late.

| fact | evidence |
|---|---|
| `Skill` is never in the tool surface | All 63 runs that requested `skills:"all"` journal `agentConfig.tools = ["Bash","Edit","Glob","Grep","Read","Write"]`, with zero `Skill` calls. Only two tool surfaces exist anywhere in `runs/`: that one and `["Glob","Grep","Read"]`. |
| The cause is our `tools` whitelist, not the SDK | Live probe (SDK 0.3.287, bundled CLI 2.1.287, sabado cwd, `skills:"all"`). With `tools` = the 6 built-ins, the model reports **"NONE"**. With `"Skill"` added to `tools`, it lists all 5 sabado skills. The SDK note "you do not need to add `'Skill'` to `allowedTools`" is about `allowedTools`. `tools` is a separate whitelist of the built-in tools, and `Skill` is one of those. |
| Reviewers are inert too | `REVIEW_TOOLS = deriveReadOnlyTools(AGENT_TOOLS)` (`review.ts:84`), so v1.15 D4 never took effect. |
| `"all"` leaks non-repo skills | Discovered on every adw-factory run: `["adwf","design","doctor","plugin-authoring"]`. Only `adwf` is a repo skill. With `Skill` enabled, the model was offered `plugin-authoring`. |
| An **explicit name list** closes that leak | Probe: `skills: ["repo-thing","adw:aws"]`. `init.skills` still lists the CLI's own discoveries, but the model reports being offered exactly those two. |
| Claude Code 2.1.287 does **not** discover `.agents/skills/` | Probe fixture with `.agents/skills/adw-cat-aws/` plus `.claude/skills/repo-thing/`: discovered `["repo-thing", …]` only. v1.15 D8's claim is false for the bundled CLI. |
| Codex does load repo skills, but they are swamped | The cqc-fe-39 session catalogue has 33 entries. The 4 repo skills cost ≈ **311 tokens**; the operator's 24 global AWS skills (`~/.agents/skills`) cost ≈ **4,558 tokens**. The other 5 are Codex's own built-in skills. |

**Cost of turning it on (measured).** Claude, sabado, 5 skills plus the `Skill` tool:
**+992 tokens** per session (cache-read 8,480 → 9,472). A skill costs one description
line until the model loads it; `SKILL.md` and `references/` are read only on demand.
The catalogue is not the cost to worry about. Wrong selection is: 33 descriptions
when 4 are relevant.

## 2. Model

```
                       always in the catalogue         loaded only on demand
target repo skills  →  one line per skill               SKILL.md → references/ → scripts/
global skills       →  one line per CATEGORY            category SKILL.md → sub-skill SKILL.md → …
everything else     →  nothing (explicitly closed)
```

A category is one more step of the same progressive disclosure that skills already use.
Nothing is chosen per ticket and no classifier runs. The model routes itself at the
moment the work actually touches the domain. Per-ticket pre-selection was considered and
rejected (D7).

## 3. Decisions

**D1: target repo skills are on by default** (reverses v1.15 D1).

```
targets/<name>.json  "skills": absent        → every skill the worktree carries (resolved to an explicit list)
                               ["a","b"]     → only those (each must exist, else refuse before dispatch)
                               []            → off
                               "all"         → refused at load, with a message pointing to absent / a list
```

Resolution is a pure function over the worktree's skill directories, run after
`provision` (the worktree is cut from `origin/<base>`, so only committed skills count).
The SDK receives the resolved **explicit list**, never the string `"all"`, because the
list is what closes the leak (§1). The operator called repo skills "custom made, and so
important", so they are never opt-in again.

**D2: `Skill` joins the tool surface exactly when the resolved list is non-empty.**
This applies to `AGENT_TOOLS` for every agent node, and to the review read-only set
(`Skill` only reads instructions, so it is review-safe). An empty list leaves the tool
surface byte-identical to today. Correction to v1.15 D5: it is the `tools` whitelist,
not `allowedTools`, that controls this.

**D3: Claude also sees a repo's `.agents/skills/`.** Claude Code does not discover that
folder (§1), but it is the canonical location v1.15 D8 chose. The factory therefore
passes the worktree's `.agents/skills/` as a per-run local plugin through the SDK's
`plugins` option. Those skills are namespaced, and the resolved list carries the
plugin-qualified names. A skill present in both `.claude/skills/` and `.agents/skills/`
(the symlink layout from D8) is offered once, under the `.claude/skills` name.

**D4: global skill categories.** Global sources are, in order, `~/.claude/skills` then
`~/.agents/skills`; the first one found wins on a name collision, and the collision is
journaled. The factory repo commits `skills/categories.json`:

```json
{
  "aws": {
    "description": "Anything touching AWS: Lambda, S3, IAM, CDK, CloudFormation, Bedrock, DynamoDB, ECS, cost, observability.",
    "include": ["aws-*", "amazon-*", "launch-with-aws", "signing-in-to-aws", "setting-up-cloudwatch-observability"]
  }
}
```

- The `description` is the trigger: it must name the domain's nouns.
- A global skill that matches no category is **off**. It is listed in `run-start` as
  `skills.unclassified`, so coverage is never silently capped.
- A category whose patterns match nothing is refused at load.
- **Every global skill is classified, exactly once** (§9). A category may carry
  `"factory": false`. Such a category is never offered to a factory agent, whatever a
  target declares. It holds the skills that cannot work headless, or that should not:
  they need subagents, a browser or web MCP, a human answering questions, or they
  touch `~/.claude` or launchd.
- **Targets get every factory category by default** (operator decision, 2026-10-03).
  `"globalSkills"` absent → all categories with `factory` ≠ `false`; a list → only
  those; `[]` → none. This is decided once per target, never per ticket.

**D5: materialisation.** All of it is deterministic, happens in `provision`, and is
journaled.

1. **Snapshot.** The included sub-skill trees are copied once into
   `~/adw/skills-cache/<sha256 of their contents>/`, and the files are made read-only
   (OS-level `chmod -R a-w`). The snapshot is reused across runs while the hash holds.
   The agent therefore reads the exact bytes the journal names, and cannot edit the
   operator's global skills.
2. **Category skill.** For each category the target opted into, a generated `SKILL.md`
   carries the category description, plus one line per sub-skill: name, its own
   description, and the absolute snapshot path to its `SKILL.md`.
3. **Claude.** The category skills form a per-run local plugin `adw` (names `adw:<cat>`)
   under `runs/<runId>/skills-plugin/`. `additionalDirectories` grants the snapshot
   directory. Probed: without that grant, the headless `Read` of a sub-skill is refused.
4. **Codex.** The category skills are written to the worktree's
   `.agents/skills/adw-<cat>/`. They are hidden from git by a per-worktree exclude
   file, the same idiom as `cmc.setup.sh`. A name collision with a repo skill refuses
   the run.

**D6: Codex host suppression, verified.** `codex exec` gains:

- `-c skills.bundled.enabled=false`, which drops Codex's 5 built-in skills;
- `-c 'skills.config=[{name="<n>",enabled=false}, …]'`, one entry for every skill
  discovered in `~/.agents/skills`, `$CODEX_HOME/skills` and any `.agents/skills` in an
  ancestor of the worktree.

Probed with `codex debug prompt-input` on 0.155.1: a name entry removes exactly that
skill (34 → 33), and `bundled.enabled=false` removes 5. A **path** entry naming a
directory does nothing. The live acceptance bullet re-measures this on `codex exec`.
This reverses v1.15 D6's "accept the leak". `skip_host_skill_discovery` is no longer
needed. A `name` selector cannot tell sources apart. If a host skill shares its name
with a repo skill, the repo skill wins: the host name is not disabled, and the
collision is journaled. A `path` selector naming the exact `SKILL.md` file is
untested; if it works, it replaces name selectors.

**D7: no per-ticket picker.** It was considered and rejected:

- a miss happens before dispatch and no gate can detect it;
- whether a ticket needs a domain often shows only at plan or build time;
- it is one more classifier to keep tuned.

The category structure makes it unnecessary. If D8's data later shows a category firing
on irrelevant tickets, a filter over **categories** (≈6 entries) can be proposed then,
tuned on that data.

**D8: provenance, so the categories are tuned on evidence.** A new journal event,
`skill-use {node, skill, via}`:

- `via: "skill-tool"` comes from the Claude `Skill` tool_use input;
- `via: "read"` is any read of a `SKILL.md` under a repo skill directory, the snapshot
  or the plugin. On Claude that is a `Read` tool_use; on Codex, a command whose argv
  names such a path.

`just skills <runId>` prints them. `run-start` carries the resolved repo list, the
categories, the snapshot hash and `unclassified`.

**D9: isolation scope.** Repo skills work in every isolation kind, because they live in
the workspace. Global categories are **worktree-only** in this amendment: the snapshot
path does not exist inside a container or an E2B sandbox. A target with `globalSkills`
in container or remote mode drops them, and says so with a journaled line rather than
refusing, since repo skills still reach the agent. Codex is worktree-only already.

## 4. Requirements and tickets

| # | requirement | ticket |
|---|---|---|
| R1 | Repo skills on by default, resolved to an explicit list; `"all"` refused; `Skill` in build and review tools iff non-empty (D1, D2) | adw-skills-02 |
| R2 | Live probe as a test (skipped without credentials, like the other live suites): `Skill ∈ init.tools`, and the model-offered set equals the resolved list (D1, D2) | adw-skills-02 |
| R3 | `skill-use` event plus `just skills` (D8) | adw-skills-03 |
| R4 | Claude sees `.agents/skills/` through a per-run plugin (D3) | adw-skills-04 |
| R5 | `skills/categories.json` loader, global discovery, read-only snapshot, generated category skills, Claude plugin plus `additionalDirectories` (D4, D5.1–3) | adw-skills-05 |
| R6 | Codex host suppression plus category skills in the worktree (D5.4, D6) | adw-skills-06 |

The order is 02 → 03 → (04 ∥ 05) → 06. 03 lands before the categories so there is a
baseline of repo-skill use to compare against.

## 5. Acceptance across the track

- A sabado run on Claude journals `Skill` in `init.tools`, and its first agent node
  reports the 5 sabado skills, and only those.
- A cmc run on Codex has a catalogue of 4 repo skills plus the categories cmc opted
  into, and nothing else (measured with `codex debug prompt-input` in the worktree).
- The `cache_creation` / catalogue token delta of both runs is recorded in the PR body.

## 6. Out of scope, operator-side (target repos, not the factory)

- **clens:** `targets/clens.json` points to `.claude/skills/coding-standards/SKILL.md`,
  which is not on `origin/main`. Remove the pointer or commit the skill.
- **content-quality-checker:** `go1-explorer`, `rise360-patterns` and `scorm-qa` live in
  `.claude/skills/`, which Codex never reads. Move them to `.agents/skills/` (v1.15 D8).
- **cmc:** `cmc-design-system` is untracked on `cqc/release-1` and not in
  `cmc.setup.sh`'s `DIRS`. Commit it, or add it to `DIRS`.
- **Skill descriptions:** a skill whose description doesn't name its domain never gets
  loaded. Review the sabado and CQC descriptions once `skill-use` data exists.

## 7. Out of scope, deferred

- Global categories inside a container or remote sandbox (D9).
- Context pointers in the repair, ci-repair and rebase-resolve prompts. Those prompts
  carry none today. The gap is real but separate, so it gets its own ticket.
- A category filter (D7), only if D8's data asks for it.

## 8. Cost to expect

- Repo skills: ≈ 60–100 tokens each plus ≈ 500 for the `Skill` tool, once per session,
  cached.
- Categories: ≈ 100 tokens each.
- Codex, cmc: the catalogue drops from ≈ 5,400 to ≈ 500 tokens.

## 9. The initial taxonomy (decided by the operator, 2026-10-03)

The operator asked for **all** global skills to be organised into top-level
categories, and for every target to get every category.

Classification of the 76 global skills (`~/.claude/skills` ∪ `~/.agents/skills`; the
24 AWS skills appear in both under the same names). Every skill lands in exactly one
category, verified by script on 2026-10-03:

| category | factory | skills |
|---|---|---|
| `aws` | ✅ | `aws-*`, `amazon-*`, launch-with-aws, signing-in-to-aws, setting-up-cloudwatch-observability (24) |
| `cloudflare` | ✅ | cloudflare, cloudflare-email-service, agents-sdk, durable-objects, workers-best-practices, wrangler, sandbox-sdk, turnstile-spin (8) |
| `engineering` | ✅ | tdd, diagnosing-bugs, domain-modeling, refactor-clean, spec-driven-development, prototype, implement, fable-mode, explain-diff, cot-leakage, frontend-design (11) |
| `tooling` | ✅ | mcp-builder, writing-great-skills, api-by-hand, go1-manual-lo, video-processor (5) |
| `productivity` | ✅ | planf3, to-spec, to-tickets, wayfinder, triage, ledger-audit, documentation, pptx-creator (8) |
| `knowledge` | ✅ | ingest, screens-to-slide, zettelkasten (3) |
| `operator-only` | ❌ | adhd, deliberate, gate, code-review, skill-creator (all spawn subagents) · web-perf (Chrome MCP) · research (web) · relecture, scroll-world, grilling, grill-with-doc (ask the human) · remind (launchd) · config-gc, worktree-manager-skill, create-worktree-skill (touch `~/.claude` or the factory's own worktrees) · sitrep, handoff (session-scoped) (17) |

The descriptions written into `categories.json` are the triggers, so each one names
its domain's nouns. Cost: 6 factory categories ≈ 600 tokens per session, against
≈ 5,400 for today's accidental Codex catalogue.

**The physical layout of `~/.claude/skills` does not change.** Claude Code and Codex
only discover skills one level deep, so nesting the folders by category would hide
every skill from interactive sessions. `categories.json` is the organisation.

New global skills are not offered until they are classified. `run-start` lists them
under `unclassified`, and `just skills-unclassified` prints the list so the gap is
visible.
