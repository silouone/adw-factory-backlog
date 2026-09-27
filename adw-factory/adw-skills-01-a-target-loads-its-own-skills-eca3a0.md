---
id: adw-skills-01-a-target-loads-its-own-skills-eca3a0
type: feat
status: in-review
priority: 2
created: 2026-09-27
caps: {minutes: 180, turns: 700, stallMinutes: 20}
depends: []
attempts: [{"runId":"adw-skills-01-a-target-loads-its-own-skills-eca3a0-1790522958340","branch":"adw/adw-skills-01-a-target-loads-its-own-skills-eca3a0","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-skills-01-a-target-loads-its-own-skills-eca3a0-1790522958340/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/130","provider":"claude","model":"sonnet"}]
---
# A target loads its own skills into the agents that build it

Spec: `specs/adw-v1.15-skills.md` (**approved 2026-09-27** — was: needs operator approval before
pickup; it is the plan §5 schema amendment this ticket's R1 requires).
Source note: `ai_docs/2026-09-27-skills-in-the-factory.md` (local-only, gitignored, so not in a run worktree; the spec carries the evidence with file:line references).

> **Pickup notes, 2026-09-27:** line numbers in R2 predate today's `build.ts` changes (adw-bug-23…27, cost-06, usage-04/05, perf-06); **re-grep each `baseOptions` call site** rather than trusting them. R6 needs a live run that writes `runs/`; if the build session cannot do it, leave R6 unticked with that reason. Caps raised 120/500 → 180/700 (today's feat runs: 295–584 API calls, 79–108 min).

A target repo's committed skills are invisible to the agents that build it. Every
agent node sends `skills: []` (`src/pipeline/nodes/build.ts:1537` →
`src/live-query.ts:514`), which the SDK documents as a **context filter**: the
skills are hidden from the model's listing and the `Skill` tool rejects them
(`sdk.d.ts:1872-1894`, SDK 0.3.209). Measured: the `Skill` tool is absent from
`system/init.tools` on every run, and `grep '"name":"Skill"' runs/` matches zero
files across every banked run.

The only route a target skill has to the agent today is as a **path** in
`target.context`, rendered as a pointer and never inlined — so no
description-triggered activation, no progressive disclosure of `references/` or
`scripts/`, and nothing that was not hand-listed. On sabado, `sabado-project` is
listed and `sabado-qa` is not, so the QA harness skill is invisible to the agent
that writes the tests.

`adw-tools-01`'s cost argument for `[]` is stale: the 17 skills and 40+ slash
commands the 2026-09-21 audit found resident in every turn were the **operator's
global** set, leaking through `settingSources`, and `adw-bug-10`
(`settingSources: ['project']`) + `adw-cost-00` (`disableBundledSkills: true`)
closed that at the source. What a non-empty `skills` lists now is the target
repo's own committed skills — measured discovery on adw-factory runs:
`["adwf","doctor"]`.

## Requirements

- [ ] **R1 — the target declares its skills** (spec D1). `TargetConfig` gains
      `skills?: "all" | readonly string[]`, validated in `src/targets/loader.ts`
      with that file's usual report-every-problem shape: `"all"` or an array of
      non-empty strings, nothing else. **Absent → `[]`** — byte-identical to
      today, and the field must be added to the schema *before* any target
      config can carry it, since the loader reports every unknown key as a
      structural error. Deliberately two-state, not `allowedTools`' three-state
      (absent and `[]` are identical at the SDK seam).
- [ ] **R2 — the option is never omitted, and never omittable** (spec D3).
      `AgentQueryOptions.skills` widens to `readonly string[] | "all"` (matching
      SDK `Options.skills?: string[] | 'all'`) and keeps its
      never-conditionally-spread contract through `live-query.ts`. In
      `baseOptions` it becomes a **required positional parameter** next to
      `systemPrompt` and `effort` — for the reason that function's own doc
      comment gives for those two: a positional cannot be dropped by a call site
      passing `{}`, and an omitted `skills` is *not* "skills off" per the SDK.
      All seven `baseOptions` call sites source it from the resolved target:
      `build.ts:706`, `build.ts:1097`, `repair.ts:100`, `ci-round.ts:875`,
      `bug.ts:400`, `review.ts:503`, `review/fix-loop.ts:408`.
      A test asserts the option rides through for a target that declares
      nothing (`[]`), one that declares `"all"`, and one that declares a list.
- [ ] **R3 — the review nodes see the same resolved set** (spec D4). `REVIEW_TOOLS`
      stays read-only, so a skill reaching a reviewer is instructions it may read
      and nothing more — and a coding-standards skill is exactly what a standards
      reviewer should read. Covered by R2's call-site sweep; asserted separately
      because it is a deliberate choice, not a side effect.
- [ ] **R4 — `AGENT_TOOLS` does not gain `Skill`** (spec D5). The SDK adds the tool
      itself when the option is non-empty (`sdk.d.ts:1872`: *"the single place to
      turn skills on; you do not need to add `'Skill'` to `allowedTools`
      yourself"*). This is **documented, not measured** — see R6. If R6's probe
      shows the tool absent with a non-empty option, **stop and report**: do not
      ship a listed-but-uninvokable skill set, and do not hand-add `Skill` to
      `AGENT_TOOLS` as a workaround without an operator decision.
- [ ] **R5 — provenance in the journal** (spec D7), at a named site.
      `node-end`'s `agentConfig.skills` already carries the CLI's **discovery**
      list, set by `consumeAgentStream` from `system/init` — that is not what the
      option gates, and the requested set never passes through the stream, so it
      cannot go there. The **requested** set rides **`node-start.skills`**,
      exactly the precedent `node-start.allowedTools` set in `adw-tools-02`: a
      per-node resolved option, known at lane-construction time, stamped up front.
      Optional on the event (absent for every non-agent node and every journal
      banked before this field), so absent/`[]` stays distinguishable from "the
      sink reported nothing" — Art. VI, a sick sink never blocks a green run.
- [ ] **R6 — the linchpin, measured live.** One real run against
      `targets/adw-factory.json` with `"skills": "all"`, recorded in the PR body:
      (a) `Skill` present in `system/init.tools`; (b) the catalogue is the repo's
      own skills only (`adwf`, `doctor` — no operator-global names); (c) the
      **absolute** `cache_creation` on the first agent call, plus the runId,
      lane, model and stage it was read from. Record the number; do **not** pick
      a baseline run to diff against — the banked runs differ in lane, model and
      prompt size, so the comparison is the operator's to make. `just usage
      <runId>` is the read.
- [ ] **R7 — Codex: nothing for repo skills, the host leak separately** (spec D6).
      Codex 0.155.1 discovers `.agents/skills` (walking up from cwd),
      `~/.agents/skills` and `$CODEX_HOME/skills` itself and injects the
      instructions by default, and `codexExecArgv` passes nothing that filters
      it — so a repo skill on the base branch already reaches a Codex run, and
      R1's field is Claude-path-only there. The operator's `~/.agents/skills`
      leaks into every Codex run. Verify `--enable skip_host_skill_discovery`
      (or the `-c` equivalent) **live** before pinning it — the flag is listed
      *under development* in 0.155.1 and no September factory Codex session
      exists on this machine to measure against. If it does not work: document
      the leak in the README's "Where it can still fail" table and close this
      requirement that way. **This requirement may not block R1–R6.**
- [ ] **R8 — docs.** README "Targets: onboarding a repo" documents `skills`
      (including absent → `[]`, and that it is target-scoped, never per-ticket),
      and `.claude/skills/adwf/references/targets.md` gains the field.

## Out of scope

Named so a build agent does not drift into them — each is deferred in
`specs/adw-v1.15-skills.md` §6:

- Ticket frontmatter `skills:` (a **ticket contract change**, needs its own
  amendment section) — `adw-skills-02`.
- `{{requiredSkills}}` in the stage templates, the `provision` presence check,
  a `skill-invoked` journal event — same follow-up.
- Moving any target's `.claude/skills` → `.agents/skills` — operator-executed.
- Touching `tools`, `strictMcpConfig`, `settingSources` or
  `disableBundledSkills`. This ticket reverses `adw-tools-01` on the **skill
  surface only**.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test` — all green.
- Loader tests: `"all"`, a list, an absent field (→ `[]`), and each malformed
  shape reported with the loader's usual every-problem-at-once output.
- A build-options test asserting `skills` is present on the options object for
  all three cases and is never a conditional spread (the
  `systemPrompt`/`tools`/`effort` precedent tests are the shape to copy).
- R6's live run, with (a), (b) and (c) pasted into the PR body.
- R7 either lands the flag with its live evidence, or lands the README row.
