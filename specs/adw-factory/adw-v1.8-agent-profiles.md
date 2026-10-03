# Amendment v1.8 — an agent profile per stage, not one model per factory

> **Status:** PROPOSED 2026-09-17. Not approved; not scheduled. Written from an
> operator observation plus four measured blocked attempts: every agent stage in
> every lane runs one hardcoded model at one hardcoded reasoning effort, and the
> provider is resolved once per run — so a plan that needs frontier reasoning and
> a mechanical fix that does not are charged the same capability, and the
> reviewer that M9 just landed judges work produced by its own model class.
> **Amends:** `adw-v1-plan.md` §5 decision 3 (per-provider default model) and
> `adw-m7-01`'s run-scoped provider decision; adds no agent node *type*, so
> Gate III is **unchanged**.
> **Binding:** `constitution.md` — unchanged. See §1 on Art. II and Art. IX.

## 1. Trigger — why this is allowed, not a silent diverge

**Art. II — "not until a run demonstrates the need."** Three demonstrations, all
on disk:

**(a) A stage starved of capability blocked four consecutive times.**
`tickets/adw-m9-04-review-fix-loop.md`'s own `attempts:` array records four
blocked runs — `1789402361952`, `1789422475817`, `1789485382215`,
`1789607658815` — every one `"model":"sonnet"`, and the last one died at the
**`plan`** node. The ticket eventually landed only after the operator
intervened. One model for every stage is not a neutral default; it is a
capability floor applied to the stage that least tolerates one.

**(b) The reviewer judges its own model class.** `adw-m9-06-lane-wiring` put
`review-standards`/`review-spec` in all three lanes, reading `agentConfig` —
the *same object* the build stages read. A reviewer that shares the builder's
model shares the builder's blind spots. Cross-model review is the single
change most likely to move M9's own §6 exit metric (operator review minutes
per PR), and it is currently impossible to express.

**(c) One model, all lanes, verified.** `src/pipeline/lanes/bug.ts:72` and
`feat.ts:71` both `import { CHORE_MODEL } from "./chore"`, and all three do
`const defaultModel = deps.model ?? CHORE_MODEL` where `CHORE_MODEL = "sonnet"`.
There is no per-lane model today, let alone per-stage. Ticket `model:` exists
but applies to **every stage at once**, which is the behaviour this amendment
replaces.

**Art. IX is the reason this is a profile and not three knobs.** Fail fast with
descriptive errors, and make invalid states unrepresentable: a model slug is
meaningless outside its provider, and the two providers do not share an effort
vocabulary (Codex accepts `none|minimal|low|medium|high|xhigh`; the Claude path
takes `EffortLevel`). A free triple `(provider, model, effort)` lets an operator
write `opus` + `codex`. A named profile cannot.

## 2. Scope

**In:** a named `(provider, model, effort)` profile; a five-level resolution
chain ending at a factory default; per-stage defaults in target config;
per-ticket and per-ticket-per-stage overrides; the attempt record carrying what
each stage actually ran on.

**Out:** choosing *which* profile each stage should use (that is operator
economics, tuned from journal data, not a spec decision); a cost model or
budget enforcement; profile selection by ticket difficulty heuristics; any new
node type; any change to what a stage *does*.

## 3. The profile

A profile is a **name** bound to a triple. Names are the only thing tickets and
target configs ever reference.

| profile | provider | model | effort |
|---------|----------|-------|--------|
| `deep` | claude | *(frontier slug)* | high |
| `standard` | claude | *(mid slug)* | high |
| `cheap` | claude | *(small slug)* | low |
| `swift` | claude | *(mid slug)* | medium |
| `codex-deep` | codex | *(codex slug)* | high |

**Slugs are pinned literals, not families resolved at run time** (decision D1).
The resolved slug must be persisted on the attempt record so a CI-repair round
resumes on the same model rather than drifting — the existing
`Attempt.model` contract (`src/intake/ticket.ts:47-52`). A family→slug resolver
would make two runs of the same ticket use different models, which is not
reproducible and cannot be persisted honestly.

The table above deliberately does not name slugs. They rotate — the operator's
own verified note on the Codex slug says *"re-probe the slug before trusting it;
OpenAI rotates these."* The registry is the one place a rotation is edited.

## 4. Resolution — five levels, most specific wins

```
1. ticket, per stage      agents: {plan: deep, build: standard}
2. ticket, whole          agent: deep
3. target, lane × stage   targets/<t>.json → agents.feat.plan
4. target, lane default   targets/<t>.json → agents.feat["*"]
5. factory default        standard
```

This is the existing `model:` precedence (ticket override > provider default >
lane constant) extended, not replaced. Level 2 subsumes today's `model:`, which
stays supported as an escape hatch for a raw slug the registry does not carry.

**Stage keys are node names** (decision D4), 1:1 with what the journal already
records: `plan`, `build`, `test`, `build-test-only`, `build-fix`,
`revise-test-only`, `repair`, `review-standards`, `review-spec`, `review-fix`.
Non-agent nodes (`dispatch`, `provision`, `baseline`, `baseline-green-check`,
`red-check`, `gates`, `commit`, `push`, `open-pr`, `assemble-*`) take no
profile; naming one is an error, not a silent no-op.

**No parser change is needed.** `parseFields` splits on the first colon
precisely so inline objects survive — `caps: {minutes: 120, turns: 600}` is the
shipped precedent, and `agents: {plan: deep, build: standard}` parses under the
same rule.

## 5. Requirements

- **R1 — a profile registry.** Named profiles resolve to `(provider, model,
  effort)`. An unknown profile name is a loud refusal at dispatch, naming the
  ticket and the unknown name, never a silent fallback to the default.
- **R2 — per-stage resolution.** Each agent node receives its OWN resolved
  profile. `agentInputs()` is already per-node; the lanes stop passing one
  shared `agentConfig` to every stage.
- **R3 — effort travels with the profile.** `CLAUDE_REASONING_EFFORT` stops
  being the single source; a stage's effort comes from its profile. The value
  is still never omitted (Art. adw-perf-02: a run never inherits the operator's
  global config).
- **R4 — target config carries lane × stage defaults** (decision D3), so
  per-repo economics differ: the self-host target may run `deep` where a
  low-stakes target runs `cheap`. Mirrors the existing `target.provider` /
  `target.systemPrompt` precedent.
- **R5 — per-stage provider.** A stage's profile selects its `AgentQuery`
  binding. The CLI's single run-scoped `selectedQuery` becomes a router.
- **R6 — the attempt record becomes a per-stage map** (decision D2), recording
  what each stage actually ran on. Readers must accept BOTH shapes: every
  attempt already on disk is scalar `provider`/`model`, and a pre-provenance
  attempt has neither.
- **R7 — isolation refusal stays structural.** `provider "codex" only supports
  worktree isolation` (`src/cli.ts:296`). A profile routing a stage to Codex on
  a container/remote run must refuse **at dispatch**, before provisioning —
  never mid-run, after tokens are spent.
- **R8 — journaled.** Each agent `node-start`/`node-end` carries the resolved
  profile name and its triple in the typed details bag. "What did this stage run
  on, and what did it cost" must be answerable from artifacts alone (Art. VI).
- **R9 — no silent capability downgrade.** If resolution cannot produce a
  profile for an agent stage, the run refuses. It never falls back to a cheaper
  model quietly.

## 6. Decisions the operator has made

- **D1 — profiles pin exact slugs**; no family→slug indirection. Rationale: the
  resolved model must persist on the attempt for resume (§3).
- **D2 — the attempt record becomes a per-stage map.** Most faithful; requires
  the reader to handle the legacy scalar shape.
- **D3 — lane × stage defaults live in target config JSON**, not code
  constants, so economics vary per repo.
- **D4 — stage keys are node names**, not coarse phases — `review-spec` is
  tunable independently of `review-standards`.

## 7. Success metrics

- A `feat` run shows **different models on different stages** in one journal.
- M9's §6 metric under cross-model review: operator review minutes per PR fall
  further than with same-model review. If they do not, the reviewer profile is
  not worth its premium and should drop to the build profile.
- A stage that blocked repeatedly on the floor model (the §1(a) case) completes
  when given a deeper profile. If capability was never the constraint, this
  amendment bought nothing and should be reverted.

## 8. Explicit non-goals

Automatic profile selection from ticket difficulty; a token budget or spend cap
(run-wide `caps:` already bounds minutes/turns); per-stage `systemPrompt` (still
per-target); profiles for non-agent nodes.

## 9. Open question this spec does not answer

**Does `ci-round` get a profile?** It has its own `CI_MODEL = "sonnet"`
(`src/pipeline/nodes/ci-round.ts:239`) and resumes an existing session rather
than starting one. Resuming a session on a different model than it was created
with is a question about the provider's resume semantics, not about this
design. Left open deliberately; `ci-round` keeps `CI_MODEL` until answered.
