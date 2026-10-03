# Amendment v1.15 — a target's own skills reach the agent that builds it

Status: **approved** by the operator, 2026-09-27.
**Partly superseded by `adw-v1.18-skill-routing.md`** (2026-10-03): D1, D5, D6 and D8's
`.agents/skills` claim are reversed there. D5 was false, and the feature shipped inert.
Source note: `ai_docs/2026-09-27-skills-in-the-factory.md` (measured from the code, the
SDK type definitions, the Codex binary and banked run journals on this machine).

## 1. Trigger — why this is an amendment, not a silent fix

`adw-v1-plan.md` §5 defines the target config, and the shipped schema
(`src/targets/loader.ts`) reports **every unknown field as a structural error** — a
target config cannot gain a key without the schema gaining it first. This amendment adds
one: `skills`. Under `CLAUDE.md`'s amendment rule the schema change is recorded here
before any red test, the same way `provider` cites "plan §5 amendment #7, decision 1" and
`agents` cites `adw-v1.8-agent-profiles.md`.

It also reverses, for the skill surface only, a decision already documented in source:
`adw-tools-01` pinned `skills: []` in `baseOptions` and wrote the rationale into the
`AgentQueryOptions.skills` doc comment. That comment stays true about the *mechanism* and
becomes stale about the *cost argument* (§2). The reversal is scoped: `tools`,
`strictMcpConfig`, `settingSources: ['project']` and `disableBundledSkills: true` are
untouched.

Nothing here changes the ticket contract. The ticket-level pin is §6, deferred.

## 2. Problem statement

A target repo's committed skills are invisible to the agents that build it.

| fact | site | evidence |
|---|---|---|
| Every agent node sends `skills: []` | `src/pipeline/nodes/build.ts:1537` → `src/live-query.ts:514` | ticket adw-tools-01 |
| `[]` is a context filter: skills are hidden from the listing and the `Skill` tool rejects them; the files stay readable via `Read`/`Bash` | SDK 0.3.209 `sdk.d.ts:1872-1894` | the SDK's own doc comment |
| The `Skill` tool is absent from the resolved surface | `system/init.tools` on every run | `["Bash","Edit","Glob","Grep","Read","Write"]` |
| Zero `Skill` invocations in every banked run | `grep '"name":"Skill"' runs/` | 0 files |
| The only path a target skill has to the agent is a **path** in `target.context`, rendered as a pointer, never inlined | `renderContextPointers`, `src/pipeline/nodes/assemble-prompt.ts` | sabado-24: the agent `Read` SKILL.md, BACKEND.md, FRONTEND.md twice each |

**Why the original cost argument is stale.** `adw-tools-01` measured a 66% cut in
cache-creation tokens from pinning `tools` + `strictMcpConfig` + `skills: []` *together*,
and the 2026-09-21 token audit then found 17 skills and 40+ slash commands — the
**operator's global** set, leaking through `settingSources` — resident in every turn at
~7% of spend. `adw-bug-10` (`settingSources: ['project']`) and `adw-cost-00`
(`disableBundledSkills: true`) have since closed that leak at the source. What a
non-empty `skills` would list today is the target repo's own committed skills, one
name-and-description line each, plus the `Skill` tool schema (measured discovery on
adw-factory runs: `["adwf","doctor"]`).

**What the pointer-only path loses:** description-triggered activation (the whole point
of a skill), progressive disclosure of `references/` and `scripts/`, and anything not
hand-listed in `target.context` — `sabado-project` is listed, `sabado-qa` is not, so the
QA harness skill is invisible to the agent that writes the tests.

## 3. Decisions

**D1 — the target declares its skills; absent means unchanged.**

```
targets/<name>.json   "skills": "all" | ["name", ...]        (optional)
absent  →  []                                                 (today's behaviour)
```

`"all"` = every skill discovered in the cut worktree. With `settingSources: ['project']`
and `disableBundledSkills: true` already pinned, discovery is the target repo's own
committed skills and nothing else. A list restricts to the named skills.

**Absent → `[]`, operator-decided 2026-09-27.** Every other optional field in
`TargetConfig` (`sandbox`, `provider`, `systemPrompt`, `agents`, `github`, `bashTimeoutMs`,
`Gate.scopedCmd`) carries an explicit "absent → byte-identical to before this ticket"
clause; that is the governing idiom for this shape of change, and it honours
`adw-bug-10`'s lesson that whatever is not explicitly declared eventually leaks. Default
`"all"` was the source note's preference and was rejected: it would change adw-factory's
and sabado's next run with no config edit. Opting in costs one line per target.

Deliberately **two**-state, not the three-state `allowedTools` shape (`adw-tools-02`,
where present-but-empty means "the target declared the field" and changes prompt
rendering). Absent and `[]` are behaviourally identical at the SDK seam, so no
distinction is manufactured.

**D2 — target-scoped, run-scoped, never per-ticket.** Mirrors `provider` and
`systemPrompt`. The ticket-level pin is a separate contract change (§6).

**D3 — `skills` stays never-omitted, and becomes a parameter, not an `extras` key.**
`AgentQueryOptions.skills` widens to `readonly string[] | "all"` (matching SDK
`Options.skills?: string[] | 'all'`) and keeps the required, never-conditionally-spread
contract it has today. In `baseOptions` it becomes a **required positional parameter**
alongside `systemPrompt` and `effort`, for the reason that function's own doc comment
already gives for those two: a positional cannot be omitted by a call site passing `{}`.
An optional `extras` key could regress to the SDK default, which the SDK documents is
**not** "skills off".

**D4 — the review nodes see the same resolved set.** `REVIEW_TOOLS` is read-only, so a
skill reaching a reviewer is instructions a reviewer may read and nothing more — and a
coding-standards skill is exactly what a standards reviewer should read.

**D5 — the SDK adds the `Skill` tool itself.** `sdk.d.ts:1872`: *"This is the single
place to turn skills on; you do not need to add `'Skill'` to `allowedTools` yourself."*
`AGENT_TOOLS` therefore does not gain `Skill`. This is **documented, not measured** —
the only measured direction is that `[]` removes the tool. It is the linchpin of the
whole change, so a live probe asserting `Skill` in `system/init.tools` is a hard
acceptance bullet, not a nice-to-have. If it is false the feature is inert and the
implementation must stop and report rather than ship a listed-but-uninvokable set.

**D6 — Codex needs nothing for repo skills, and its host leak is a separate,
separately-failable concern.** Codex CLI 0.155.1 discovers skills itself from
`.agents/skills` (walking up from cwd), `~/.agents/skills` and `$CODEX_HOME/skills`, and
injects their instructions by default (`"skills":{"includeInstructions":true}` on every
session). `src/codex-query.ts`'s argv passes nothing that filters this, so a repo skill
on the base branch already reaches a Codex run. Two consequences:

- Repo skills: no code change. D1's `skills` field is a **Claude-path** option; on Codex
  it is provenance only.
- The host leak (the operator's `~/.agents/skills`) is the same class of leak the Claude
  path closed with `settingSources`. Codex exposes `skip_host_skill_discovery`, listed
  *under development* in 0.155.1, and no September factory Codex session exists on this
  machine to measure against. It must be verified live before it is pinned; if it does
  not work, the documented fallback is to record the leak in the README's honest-edges
  table. Closing it is **not** allowed to block D1.

**D7 — provenance.** The journal already carries `agentConfig.skills` — the CLI's
**discovery** list, which is not what the option gates. A run must also be able to say
which set was *requested*. The resolved value rides the same journal shape, so
`just usage` and the run screen can show "these skills were on the table".

**D8 — `.agents/skills/` is the canonical location in a target.** It is the one directory
both providers read (Claude Code 2.1.283 reads `.claude/skills/` and `.agents/skills/`;
Codex 0.155.1 reads only `.agents/skills/`). Recommendation for every target:
`.agents/skills/` canonical, `.claude/skills` a symlink to it for interactive use. This
is operator-executed, target-side work and is out of scope here (§6).

## 4. Requirements this amendment authorises

| # | requirement | ticket |
|---|---|---|
| R1 | `TargetConfig.skills?: "all" \| readonly string[]`, validated in `loader.ts`; absent → `[]` | adw-skills-01 |
| R2 | `AgentQueryOptions.skills` widened and never omitted; `baseOptions` takes it positionally; every call site sources it from the resolved target | adw-skills-01 |
| R3 | Review nodes resolve the same set (D4) | adw-skills-01 |
| R4 | Codex host-discovery flag, verified live or documented as a known leak (D6) | adw-skills-01 |
| R5 | The requested set is journaled next to the discovered one (D7) | adw-skills-01 |
| R6 | A live probe: `Skill` in `init.tools`, catalogue = the repo's skills only, `cache_creation` delta recorded (D5) | adw-skills-01 |

## 5. Cost to expect

One name + description line per skill (~50–100 tokens) plus the `Skill` tool schema,
once per session, cached. The number is measured on the first live run by diffing
`cache_creation` on the first call against a baseline run and belongs in the ticket's
verify block, not in prose here.

## 6. Out of scope — deferred, with reasons

- **A ticket pins the skills it needs** (`skills: [cqc-frontend]` in frontmatter,
  `{{requiredSkills}}` rendered into every stage template, a `provision` presence check,
  a `skill-invoked` journal event). This is a **ticket contract change** and needs its own
  amendment section before its own red test. Pin semantics — required, not filter —
  because a filter would throw away description-triggered self-selection and cannot be
  honoured on Codex, which has no per-run allow-list. Deliberately after D1: there is no
  point pinning a skill the agent cannot see.
- **Per-stage pins.** The ticket contract already has `agents: {plan, build}`
  (`adw-profile-01`); a named profile could carry `skills` later without a new top-level
  field.
- **`suggestSkills(ticketBody, catalog)`** — a pure scorer behind `just ticket-skills
  <id>`, prefilling `skills:` at ticket-authoring time. Premature before the pin exists.
- **The `.agents/skills/` move** (D8) on sabado, adw-factory and CMC, plus committing
  CMC's four skills, `.agents/design-system/`, `docs/cqc/` and the AGENTS.md pointer to
  `master` — all untracked today, and `provision` cuts every worktree from
  `origin/<base>` (`adw-bug-16`). That blocker is git, not the factory.

## 7. Open points for the operator

1. ~~Default `"all"` or `[]`~~ — decided 2026-09-27: `[]` (D1).
2. ~~Should the review nodes see skills~~ — decided: yes (D4).
3. ~~Codex host leak~~ — decided 2026-09-27 (orchestrator, on operator approval of this spec): **accept the host leak until `skip_host_skill_discovery` stabilises** (D6's fallback); a per-run `CODEX_HOME` is out of scope. Original question: `skip_host_skill_discovery` is "under development" in Codex 0.155.1 — accept the host
   leak until it stabilises (D6's fallback), or pin a per-run `CODEX_HOME` with no skills
   in it (heavier, deterministic, and a larger change than this amendment).

## 8. Amendment — `AGENT_ALLOWED_TOOLS` carries read-only output filters (adw-tools-03)

No dedicated adw-tools-01/02 spec section exists; this file is the only spec that
names those tickets, so the amendment lives here.

- **The allowlist includes `head`, `tail`, `grep`, `wc` and `sort`.** Claude Code
  checks each segment of a compound command, so `bunx tsc --noEmit 2>&1 | head -40`
  is a `bunx` segment (covered by `Bash(bunx:*)`) plus a `head` segment (covered by
  nothing). The `head` segment goes to the auto-mode classifier, and is denied when
  the classifier is down (adw-bug-40). The factory's own check commands therefore
  depended on the classifier. The five filters only read stdin and print to stdout.
- **Rejected: `sed`, `awk`, `xargs`, `tee`, `cat`.** `sed` (`-i`, `w`) and `tee` write
  files; `awk` can write files and run commands (`system`, `print >`); `xargs` runs
  arbitrary commands; `cat` is not a filter an agent needs after a pipe, and
  approving it lets any file be read or concatenated into a redirect.
- Merge behaviour is unchanged: a target's own `allowedTools` (adw-tools-02) is
  deduplicated onto the factory list, order preserved. The build, repair and review-fix
  prompts tell the agent to run check commands unpiped or piped only through
  `head`, `tail` or `grep`.
