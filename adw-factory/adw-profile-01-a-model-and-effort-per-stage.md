---
id: adw-profile-01-a-model-and-effort-per-stage
type: feat
status: done
priority: 1
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-profile-01-a-model-and-effort-per-stage-1789714381296","branch":"adw/adw-profile-01-a-model-and-effort-per-stage","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-profile-01-a-model-and-effort-per-stage-1789714381296/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/74","provider":"claude","model":"sonnet"}]
---
# A plan and a mechanical fix should not be charged the same capability

> `specs/adw-v1.8-agent-profiles.md` R1–R4, R8, R9, and decisions D1/D3/D4.
> The first of two: named profiles, per-stage resolution, and the whole
> registry/precedence machinery — on the **Claude path only**. Per-stage
> *provider* is `adw-profile-02`, which depends on this.

## Context

Every agent stage in every lane runs one model at one effort. Verified:

- `src/pipeline/lanes/chore.ts:79` — `export const CHORE_MODEL = "sonnet"`.
- `src/pipeline/lanes/bug.ts:72` and `feat.ts:71` both
  `import { CHORE_MODEL } from "./chore"`; all three then do
  `const defaultModel = deps.model ?? CHORE_MODEL`.
- `src/pipeline/nodes/build.ts:271` — `CLAUDE_REASONING_EFFORT = "high"`, one
  call site (`build.ts:710`), applied to every stage.
- Ticket `model:` overrides — but for **every stage at once**.

The need is measured: `tickets/adw-m9-04-review-fix-loop.md`'s `attempts:`
array records four blocked runs, all `"model":"sonnet"`, the last dying at the
`plan` node.

**The seam already exists.** `makeBuildNode(query, config, instance)` takes
`AgentNodeConfig` **per node** — its own doc comment says *"Per-node lane
configuration (plan decision 10)"* — and `agentInputs(ctx, config)`
(`build.ts:634`) already resolves model per node. The lanes simply construct
one `agentConfig` and hand the same object to every stage. This ticket stops
that.

## Deliverables

A profile registry module; `src/targets/` config schema + validation;
`src/intake/ticket.ts` frontmatter parsing; per-stage config construction in
`src/pipeline/lanes/{chore,bug,feat,shared}.ts`; `build.ts`'s effort source;
and the tests for each.

## Requirements

- [ ] **R1 — registry.** Named profiles → `(provider, model, effort)`. Ship
      `deep` / `standard` / `cheap` on the Claude path. Slugs are pinned
      literals (D1). An unknown profile name is a loud refusal at dispatch
      naming the ticket and the unknown name (Art. IX) — never a silent
      fallback.
- [ ] **R2 — per-stage resolution.** Each agent node gets its OWN resolved
      profile. The five-level chain, most specific first: ticket-per-stage →
      ticket-whole → target lane×stage → target lane `"*"` → factory default.
- [ ] **D4 — stage keys are node names**: `plan`, `build`, `test`,
      `build-test-only`, `build-fix`, `revise-test-only`, `repair`,
      `review-standards`, `review-spec`, `review-fix`. A key naming a
      non-agent node (`gates`, `commit`, `push`, `open-pr`, `dispatch`,
      `provision`, `baseline`, `baseline-green-check`, `red-check`,
      `assemble-*`) is a validation ERROR, not a silent no-op.
- [ ] **R3 — effort travels with the profile.** `CLAUDE_REASONING_EFFORT` stops
      being the single source. Effort is still NEVER omitted from the SDK
      options — adw-perf-02's guarantee (a run never inherits the operator's
      `~/.claude` config) must survive, and its test must stay green.
- [ ] **R4 — target config** gains the lane × stage default map (D3), validated
      like the existing `provider`/`systemPrompt` fields. Absent → the factory
      default, byte-identical to today's behaviour.
- [ ] **Ticket frontmatter:** `agent: <profile>` (whole ticket) and
      `agents: {plan: deep, build: standard}` (per stage). NO parser change —
      `parseFields` splits on the first colon so inline objects survive;
      `caps: {minutes: 120, turns: 600}` is the precedent. A malformed value is
      a reported error, never a silent default.
- [ ] **Back-compat:** existing `model:` keeps working as a raw-slug escape
      hatch, at precedence level 2 (ticket-whole). A ticket carrying BOTH
      `model:` and `agent:` is an error — two answers to one question.
- [ ] **R8 — journaled.** Each agent `node-start`/`node-end` carries the
      resolved profile name and its triple in the typed details bag (Art. VI),
      the way the review nodes carry their verdict.
- [ ] **R9 — no silent downgrade.** If resolution cannot produce a profile for
      an agent stage, the run refuses rather than falling back to a cheaper
      model.
- [ ] **Unchanged:** `provider` stays run-scoped (that is `adw-profile-02`);
      `systemPrompt` stays per-target; `ci-round`'s `CI_MODEL` is untouched
      (spec §9 open question).

## Verify

- [ ] Red tests first, per the two non-negotiables. At minimum:
      a `feat` run where `plan`, `build` and `review-spec` each receive a
      DIFFERENT model+effort, asserted on the options object handed to the fake
      query (the same seam `adw-bug-10`'s test used — the only layer where this
      is a genuine observation rather than a proxy);
      each precedence level overriding the one below it;
      an unknown profile name refusing at dispatch;
      a non-agent node name in `agents:` rejected;
      `model:` + `agent:` together rejected;
      absent config producing options byte-identical to today.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] The adw-perf-02 effort test still green — effort never omitted.

## Out of scope

Per-stage provider and the query router (`adw-profile-02`); the attempt-record
per-stage map (R6, also `adw-profile-02` — it only becomes observable once
providers can differ); choosing which profile each stage should use (operator
economics, tuned from journal data); `ci-round`.
