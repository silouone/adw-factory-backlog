# Amendment v1.16: model routing (a gear for every agent node, chosen by code)

> **Status:** PROPOSED 2026-10-03 from an operator session, `ready-for-agent`
> once approved. Decisions D1–D9 below were made by the operator in that
> session.
> **Amends:** `adw-v1.8-agent-profiles.md`:
> - §8 non-goal *"automatic profile selection from ticket difficulty"* is
>   lifted, constrained to a **deterministic pure function** with a shadow
>   period;
> - §9 (does `ci-round` get a profile) is answered: **yes**;
> - the profile table is replaced by the gear ladder.
>
> Also amends the README's "router-less" line: there is still no router
> **agent**.
> **Binding:** `constitution.md` is unchanged. No new agent node *type*, so
> Gate III is unchanged.
> **Evidence:** `ai_docs/2026-10-03-model-routing-assessment.md`, recomputed
> from every banked journal (383 runs, 1,538 agent node-ends).

## Problem Statement

The operator pays the same capability for every agent node in every lane. A
plan that needs frontier reasoning and a lint fix that needs none both run on
Sonnet 5.5 at `high` effort (Claude path) or gpt-6-sol at `high` (Codex path).
This is not a missing feature. On the Claude path, per-stage profiles were
built in `adw-profile-01`/`-03` and have **never been selected once**: 1,538 of
1,538 agent node-ends ran one gear. No target sets `agents` and no ticket sets
`agent:` or `agents:`. On the Codex path (9 of 14 targets) the mechanism does
not exist: every stage gets one placeholder profile, and reasoning effort is
hardcoded in argv.

The operator feels this three ways:

1. **Speed.** Build p50 is 14.1 min (p90 36.2), review-fix 12.5, plan 7.4,
   test 6.8. Mechanical stages run at full thinking effort.
2. **Headroom.** Claude runs bill against a subscription with 5-hour and
   weekly windows, plus a separate per-model weekly bucket. Codex is an
   unmetered seat, so there wall-clock is the only currency. Every unneeded
   high-effort turn is spent headroom.
3. **Quality where it matters.** Plan and review, the judgement seams, get no
   more capability than a dependency bump. The most expensive failure in the
   factory is a blocked run (the whole run is lost), and nothing climbs a gear
   when a stage is visibly struggling.

The operator also cannot read the trade-off: the rate card is stale (Sonnet
priced 3/15 vs 2/10 today, Opus 15/75 vs 4/20), so the web view overstates
Claude spend by about 1.5× and makes Opus look 5× too expensive.

## Solution

Every agent node, in every lane and in the post-PR rounds, runs on a **gear**:
a pinned (provider-specific model, effort) pair from one seven-step ladder. The
gear is chosen **by code** from four inputs:

1. a **base table** (lane × node);
2. the ticket's **size** (derived from the ticket) or a declared **extreme**;
3. **in-run escalation**: the last allowed round of a bounded loop runs one
   gear up, inside the existing round budget;
4. **cross-run escalation**: a stage that blocked last attempt starts one gear
   up.

The choice and its reasons are journaled for every node.

It ships as eight stacked tickets (*Phases → tickets* below), and each step earns the next:

- **Foundation:** honest prices, the gear ladder, every agent node routable, Codex gears, and the A/B report.
- **The router with the base table:** per-target `routing: off | shadow | live`, journaled.
- **Escalation**, in-run and cross-run.
- **Size and declared extreme**, in shadow first.

Live A/B runs and go-live flips are operator steps, not tickets (D10).

From the operator's side: they see in the journal, the run view and the PR
body which gear each node ran on and why. They can override any node per
ticket or per target exactly as today. They declare a ticket `extreme` when it
warrants Fable. Without touching anything, they get faster mechanical stages
and stronger plans and reviews.

## The gear ladder (D1–D3)

| gear | Claude | Codex | intent |
|---|---|---|---|
| **G0** flash | Haiku 4.5 · thinking budget *medium*¹ | gpt-6-luna · medium | mechanical, checked afterwards by a deterministic node |
| **G0+** balanced | Haiku 4.5 · thinking budget *high*¹ | gpt-5.6-terra · medium | the same work with a safer floor |
| **G1** swift | Sonnet 5.5 · medium | gpt-6-sol · medium | routine work against a clear plan |
| **G2** standard | Sonnet 5.5 · high | gpt-6-sol · high | today's default for everything |
| **G3** deep | Opus 5.5 · high | gpt-6-astra · high | judgement: plan, review |
| **G4** max | Opus 5.5 · xhigh | gpt-6-astra · xhigh | escalation and large tickets |
| **G5** frontier | Fable 5.1 · high | gpt-6-astra · max | extreme tickets only |

¹ **Haiku 4.5 does not support `effort`.** Its model page lists default effort as
"Not supported", and Claude Code's effort table omits it. It uses manual extended
thinking (`thinking: {type: "enabled", budgetTokens: N}`) instead. Probed
2026-10-03 on the bundled CLI 2.1.287: `--effort medium` is accepted without
error, but the model's thinking is not governed by it. So a Claude G0/G0+ gear
carries a **thinking budget**, not an effort. The two budgets are named
constants (proposed: 8k for *medium*, 24k for *high*), and the gear type makes
"effort on Haiku" unrepresentable.

**Fable 5.1 is runnable headless.** Probed 2026-10-03 on the bundled CLI
2.1.287: `claude-fable-5-1` returns a clean turn (`is_error:false`). The
minimum CLI version for G5 is therefore ≤ 2.1.287.

Model ids are pinned exact slugs (v1.8 D1 stands): `claude-haiku-4-5-20251001`,
`claude-sonnet-5-5`, `claude-opus-5-5`, `claude-fable-5-1`, and the Codex
slugs as listed in the catalog.

## The routing table (D4)

Columns are ticket size; **extreme** is declared by the operator. "esc" is the
in-run escalation rule. Deterministic nodes take no gear and are omitted.
**Today, every cell is G2.**

**chore**

| node | small | normal | large | extreme | esc |
|---|---|---|---|---|---|
| build | G0 | G1 | G2 | G3 | cross-run +1; extreme → G5 |
| repair | G0 | G1 | G2 | G3 | round 3 of 3: +1 |
| review-standards | G1 | G2 | G3 | G3 | n/a |
| review-spec | G1 | G2 | G3 | G5 | n/a |
| review-fix | G1 | G2 | G2 | G3 | round 2 of 2: +1 |

**feat**

| node | small | normal | large | extreme | esc |
|---|---|---|---|---|---|
| plan | G3 | G3 | G4 | G5 | cross-run +1 |
| build | G0+ | G1 | G2 | G3 | cross-run +1; extreme → G5 |
| test | G0+ | G1 | G2 | G2 | cross-run +1 |
| repair | G0+ | G1 | G2 | G3 | round 3 of 3: +1 |
| review-standards | G3 | G3 | G3 | G3 | n/a |
| review-spec | G3 | G3 | G4 | G5 | n/a |
| review-fix | G2 | G2 | G2 | G3 | round 2 of 2: +1 |

**bug**

| node | small | normal | large | extreme | esc |
|---|---|---|---|---|---|
| plan | G3 | G3 | G4 | G5 | cross-run +1 |
| build-test-only | G0+ | G1 | G1 | G2 | n/a |
| revise-test-only | G0+ | G1 | G1 | G2 | round 2 of 2: +1 |
| build-fix | G1 | G2 | G3 | G3 | cross-run +1; extreme → G5 |
| repair | G0+ | G1 | G2 | G3 | round 3 of 3: +1 |
| review-standards | G3 | G3 | G3 | G3 | n/a |
| review-spec | G3 | G3 | G4 | G5 | n/a |
| review-fix | G2 | G2 | G2 | G3 | round 2 of 2: +1 |

**post-PR rounds (every lane).** These are single-shot rounds, so there is
no in-run escalation.

| node | small | normal | large | extreme |
|---|---|---|---|---|
| ci-repair | G1 | G2 | G2 | G3 |
| rebase-resolve | G1 | G2 | G3 | G3 |

These values are the **starting hypothesis**. The Phase 0 A/B can move any
build, test or repair cell before Phase 1 ships, and its result is recorded
in this amendment.

## User Stories

1. As the operator, I want every agent node to run on a gear chosen for that node, so that a plan and a lint fix are not charged the same capability.
2. As the operator, I want one ladder of seven gears shared by both providers, so that I reason about "how much capability" once instead of per provider.
3. As the operator, I want the lowest gear to use medium effort at minimum, so that no factory node ever runs at `low` effort.
4. As the operator, I want gpt-5.6-terra available as the Codex G0+ gear, so that Codex mechanical stages have a balanced floor between luna and sol.
5. As the operator, I want Fable 5.1 as the top gear, so that the hardest tickets can plan and be reviewed at frontier capability.
6. As the operator, I want Fable used only on tickets I declare extreme, plus build escalation on those tickets, so that the scarce Fable quota is never spent by inference.
7. As the operator, I want to declare a ticket extreme in its frontmatter, so that the routing decision for rare, expensive work stays mine.
8. As the operator, I want plan and both reviews to never fall below G3 in feat and bug, so that the judgement seams are never starved by a size heuristic.
9. As the operator, I want feat and bug build stages to never fall below G0+, so that only chore work can reach the G0 floor.
10. As the operator, I want a small ticket's build, test and repair to drop one gear, so that mechanical tickets finish faster and spend less headroom.
11. As the operator, I want a large ticket's plan and build to rise one gear, so that big tickets are not lost to an under-powered stage.
12. As the operator, I want ticket size derived deterministically from the ticket file, so that two dispatches of the same ticket route identically.
13. As the operator, I want the final round of a bounded loop (repair round 3, revise round 2, review-fix round 2) to run one gear up, so that a struggling stage gets more capability before the run blocks.
14. As the operator, I want in-run escalation to stay inside the existing round budget, so that bounded autonomy is unchanged and no extra round is ever added.
15. As the operator, I want a stage that blocked on the previous attempt to start one gear up on the next dispatch, so that re-dispatching a blocked ticket is not a repeat of the same failure.
16. As the operator, I want cross-run escalation to ignore non-capability blocks (approval outage, provision failure, red base, dispatch refusal), so that an infrastructure failure never inflates a gear.
17. As the operator, I want gears to only ever go up through escalation, so that the no-silent-downgrade rule (v1.8 R9) still holds.
18. As the operator, I want the ladder clamped at G0 and G5, so that escalation from the top gear is a no-op and is journaled as such.
19. As the operator, I want an explicit per-stage pin in a ticket to never be adjusted by the router or escalation, so that I can pin any node by hand exactly as today.
20. As the operator, I want target-level `agents` to act as the router's base table, so that per-repo economics stay possible and still benefit from size, extreme and escalation.
21. As the operator, I want the 87 tickets pinning `model: gpt-5.6-sol` to keep their current behaviour, so that this track doesn't silently change the CQC/CMC runs.
22. As the operator, I want a journal event per agent node naming its gear, model, effort and the reason the router chose it, so that "why did this node run on Opus" is answerable from the journal alone.
23. As the operator, I want the router's decision journaled once at dispatch as a full per-node map, so that I can read the run's whole routing plan in one line.
24. As the operator, I want a shadow mode where the router's pick is journaled next to the gear that actually ran, so that I can judge the router before it drives anything.
25. As the operator, I want shadow mode to be the default when the router first ships, so that turning routing on is a deliberate act.
26. As the operator, I want a report comparing shadow picks against outcomes, turns, tokens and wall-clock across runs, so that the decision to go live is made from data.
27. As the operator, I want the attempt record to carry what each node actually ran on, so that a CI or rebase round resumes on a known gear.
28. As the operator, I want ci-repair and rebase-resolve to be routable stages, so that no agent node bypasses the routing system.
29. As the operator, I want Codex nodes to honour a per-stage model and effort, so that the 9 Codex targets get routing too.
30. As the operator, I want Codex effort passed per stage, not baked into argv, so that a Codex G1 node really runs at medium.
31. As the operator, I want the rate card to match today's published prices for every gear model, so that the web view and the A/B report are honest.
32. As the operator, I want every gear model priced (including gpt-6-luna, gpt-reserve, gpt-5.6-terra and Fable 5.1), so that no routed run shows as unpriced.
33. As the operator, I want a run refused at dispatch when a resolved gear's model is not runnable on the installed SDK or CLI, so that a gear never fails mid-run after tokens are spent.
34. As the operator, I want a one-shot probe confirming Haiku 4.5 honours `effort`, so that G0 and G0+ are known to be distinct gears before they exist.
35. As the operator, I want a one-shot probe confirming Fable 5.1 runs through the bundled SDK in headless mode on the subscription, so that G5 is known to work before a ticket depends on it.
36. As the operator, I want the A/B run before the base table ships (standard vs swift on one feat ticket, flash vs balanced on one chore per provider), so that the base values come from measurement, not belief.
37. As the operator, I want the A/B report to show turns, total tokens, API-equivalent cost, wall-clock and outcome side by side, so that I can see whether a lower gear's extra turns cancel its saving.
38. As the operator, I want the run view to show each node's gear, so that a glance at a run tells me where capability was spent.
39. As the operator, I want the PR body to list the gears used, so that a reviewer knows what produced the code.
40. As the operator, I want Opus and Fable consumption visible against the weekly per-model bucket before dispatch, so that routing doesn't exhaust a scarce quota unnoticed.
41. As the operator, I want an unknown gear name in a ticket or target refused at parse or load time, naming the ticket and the value, so that a typo never silently falls back to a default.
42. As the operator, I want the `cheap` profile (haiku · low) retired, so that the registry cannot express a gear below the agreed floor.
43. As the operator, I want existing profile names (`deep`, `standard`, `swift`) to keep resolving during migration, so that nothing configured today breaks.
44. As a builder agent picking up a ticket from this track, I want each ticket to name its phase and its red test, so that Article I is followed without interpretation.
45. As a reviewer, I want routed runs to stay byte-identical in behaviour to today when routing is off, so that turning the feature off is a real off switch.

## Implementation Decisions

### Operator decisions (made 2026-10-03)

- **D1, the floor:** Haiku 4.5 or gpt-6-luna at **medium**. `low` effort is never used by the factory. The `cheap` profile is retired, not kept beside the floor.
- **D2, terra:** gpt-5.6-terra is the Codex G0+. If the Phase 0 A/B shows luna·medium ≥ terra·medium on the same ticket, terra is dropped and G0+ becomes luna·high.
- **D3, Fable:** Fable 5.1 is G5. It is reached only by (a) a declared `extreme` ticket's plan and review-spec, or (b) build-family escalation on an extreme ticket. Fable quota is the binding constraint, not price.
- **D4, the table:** the routing table above is the starting hypothesis. Its build, test and repair cells are confirmed or corrected by the Phase 0 A/B before Phase 1.
- **D5, the router is code:** a pure, deterministic, total function. No model call sits in the routing path. A model-based sensor (for example the TypeSafe/Jev difficulty judgment) is out of scope here. If ever added, it is one more input to the same pure function and is shadow-compared against the deterministic size rule first.
- **D6, A/B before routing:** Phase 0's measurement gates Phase 1. `adw-cost-02` R3/R3a (left unchecked on a done ticket) is folded into Phase 0.
- **D7, shadow first:** the size and extreme router ships journaling only. Going live is an operator decision recorded as an amendment note, after enough shadow runs (target: 30 routed dispatches across both providers).
- **D8, the 87 `gpt-5.6-sol` pins stay.** They remain whole-ticket overrides. Migrating CQC/CMC to target-level `agents` is a separate later decision, not part of this track.
- **D9, escalation never adds rounds.** It changes the gear of a round that already exists.
- **D10, no manual tickets.** Every ticket in this track is agent-executable. The few steps that need live runs or a decision are operator steps listed under *Phases*, not tickets. Probes that could be run in-session were run (see the ¹ note).
- **D11, no separate Phase 1 config.** The built-in base table *is* the router's level-5 default. Per-target routing mode (`off | shadow | live`) replaces hand-written `agents` blocks as the way a target adopts the table. `agents` stays the per-target override.

### Modules

- **Gear registry** (replaces the profile registry's table; same leaf-module constraints, so no imports from intake or targets). It holds a gear id, plus per-provider `{model, effort}`, plus legacy name aliases: `deep`→G3, `standard`→G2, `swift`→G1. The effort vocabulary is typed per provider (Claude: low…max; Codex: low…ultra) so an invalid pair cannot be represented.
- **Router**, the one new deep module. It is a pure function from (lane, ticket, target config, attempt history, provider) to a per-node plan. For each agent node the plan holds the gear, the resolved model and effort, the gear for each round of that node's bounded loop, and a reason list. How it composes with the existing five-level chain:
  - **Ticket-level settings (v1.8 levels 1–2) are absolute.** A per-stage `agents:` pin is never adjusted. A whole-ticket `agent:`/`model:` is the base for every node, and escalation may still raise it (never lower it).
  - **Target-level `agents` (levels 3–4) is the base table.** Phase 1 writes the table there, and the router reads it as its lane × node base rather than being shadowed by it.
  - **The built-in table is the factory default (level 5)** for targets that set none.

  So size, extreme and escalation apply on top of any base except a level-1 pin. The router owns:
  - **base-table lookup** (lane × node);
  - **size derivation**: a pure function of the ticket's requirement-checkbox count, body length, distinct paths named, and declared `caps`. Thresholds are constants set from the shadow data; initial values are proposed in the Phase 4 ticket;
  - **extreme** from frontmatter;
  - **cross-run escalation** from the `attempts` history. Only blocked attempts whose reason is classified as capability-shaped count. The classifier is a pure function of the attempt's recorded block reason;
  - **floors and clamps**;
  - a **mode**: `off` (today's behaviour exactly), `shadow` (compute and journal, run the resolved-chain gear) or `live`.
- **Agent stage names** gain `ci-repair` and `rebase-resolve`. Both post-PR rounds read their gear from the router, and the hardcoded CI model constant goes away.
- **Engine and round integration.** A bounded loop's retry target receives the gear for **its round number**. The engine already owns the round counter, so the node reads its gear from the plan by round. No node counts rounds.
- **Codex query binding** stops hardcoding effort and reads the per-stage effort from the agent query options. The run-scoped provider stays; a per-stage provider is still `adw-profile-02`'s job and is out of scope here.
- **Ticket contract** gains optional `complexity: extreme` and accepts gear ids (`G0`…`G5`) as well as the legacy profile names in `agent:`/`agents:`. Unknown values are refused at parse time with the ticket id and the value.
- **Target config** accepts gear ids in `agents`, and gains an optional `routing: off | shadow | live` (default `off` until Phase 4 ships, then `shadow`).
- **Journal:**
  - a new `route` event at dispatch: the full per-node plan, mode, size, extreme, history inputs, and reasons;
  - each agent node-end adds `gear` (and `shadowGear` in shadow mode) to the existing usage bag beside model, effort and profile;
  - escalation steps journal the round, the from and to gears, and why.
- **Attempt record** becomes the per-stage map v1.8 D2 already decided. Readers accept both the legacy scalar shape and the map.
- **Rate card:**
  - updated to the published prices read 2026-10-03: Sonnet 5.5 2/10/0.20/2.50, Opus 5.5 4/20/0.20/5, Haiku 4.5 1/5/0.10/1.25, Fable 5.1 10/50/0.25/12.50;
  - priced keys for gpt-6-luna, gpt-reserve and gpt-5.6-terra (from OpenAI's published pages, dated);
  - the pinning test extends to every gear model, so a gear without a rate fails the suite.
- **Dispatch preflight** refuses a run whose resolved plan names a model the installed SDK or CLI cannot run (the Opus 5.5 ≥ 2.1.280 precedent). Minimum versions for Haiku and Fable are recorded once the Phase 0 probes run.
- **Surfaces:** the run view shows a gear chip per node, the PR body lists gears per node, and the usage panel shows Opus and Fable weekly consumption next to the dispatch action (extends v1.7).
- **A/B harness:** a script that compares two runs, or two sets of runs, of the same ticket on turns, tokens, API-equivalent cost, wall-clock and outcome. It is built on the existing run-autopsy and scorecard scripts, not a new reader.

### Phases → tickets (stacked, 8 tickets; see the backlog `adw-route-*`)

| ticket | delivers | blocked by |
|---|---|---|
| **route-01** gear ladder and honest prices | rate card to today's prices and every gear model priced; gear registry G0–G5 per provider (Haiku via thinking budget); legacy aliases; `cheap` retired; gear ids in tickets and targets; unknown values refused | none |
| **route-02** every agent node is routable, and dispatch refuses an unrunnable gear | ci-repair and rebase-resolve as stages; per-stage attempt map with blocked node and reason; preflight model-runnable refusal | 01 |
| **route-03** Codex gears | per-stage model and effort on the Codex path; placeholder profile gone | 01 |
| **route-04** A/B report | same-ticket comparison of turns, tokens, cost, wall-clock and outcome | 01 |
| **route-05** the router, base table, `route` event, modes, surfaces | pure router over the built-in table and target `agents`; `route` journal event; per-node `gear`/`shadowGear`; `routing: off/shadow/live` per target; run-view gear chip; PR-body gears | 02, 03 |
| **route-06** escalation | in-run (round-indexed gear) and cross-run (capability-shaped blocks only) | 05 |
| **route-07** size and extreme | size derivation, `complexity: extreme`, shadow-vs-outcome comparison added to the A/B report | 04, 05 |
| **route-08** Opus/Fable weekly headroom at dispatch | extends the v1.7 usage panel | 01 |

**Operator steps (not tickets, D10):**

1. Approve and commit this amendment before route-05 is dispatched.
2. After route-04 lands, run the A/B (feat G2 vs G1; chore G0 vs G0+ on Claude, luna vs terra on Codex after route-03) and record the result in the routing table above.
3. After route-05, set targets to `shadow`; set them to `live` once the A/B confirms the base table.
4. After route-07, go live on size and extreme after about 30 shadow dispatches.

## Testing Decisions

**What a good test is here.** Assert external behaviour at the highest
seam, not internals:

- given a ticket, target and history, which gear and model each node gets;
- what the agent query actually received;
- what the journal records.

Never assert how the router computes it.

**Seams.** These are proposed; the operator should confirm them.

1. **The router function**, the primary seam. Table-driven tests over (lane × node × size × extreme × history × round × provider) assert the resolved plan. One seam covers base table, floors, clamps, size, extreme and both escalations.
   *Prior art:* the existing profile-resolution and lane-shared resolution tests.
2. **The agent query boundary** (fake query). A lane run with routing on asserts each agent node's query received the planned model and effort, including the round-3 repair's escalated gear, and the Codex argv carrying per-stage effort.
   *Prior art:* the agent-node tests, the Codex query argv tests, the lane-chain tests.
3. **The journal.** The `route` event, per-node `gear` and `shadowGear`, and escalation events are present and well-formed. Routing off produces the same journal shape as today.
   *Prior art:* the CLI observability tests.

**Other tested modules:**

- the rate card: every gear model has a rate, and the published values are pinned;
- ticket parsing: `complexity`, gear ids, unknown-value refusal;
- target loading: gear ids, `routing` mode;
- the attempt reader: both legacy and map shapes;
- the capability-shaped block classifier: outage, provision, red base and refusal never escalate.

**Art. I.** Each ticket lands its red tests first, at seam 1 wherever possible.
Phase 0b and 0d are live measurements, not unit tests. Their deliverable is
recorded evidence in this amendment and the PR body.

## Out of Scope

- A per-stage **provider** (a Claude builder with a Codex reviewer): that is `adw-profile-02`, blocked on its turn ceiling and to be re-sliced separately.
- Any model call in the routing path (LLM or Jev classifier). That's possible later as a shadow input only (D5).
- Migrating the 87 `gpt-5.6-sol` pinned tickets (D8).
- Changing what any node does, its prompts, or any round bound.
- A token budget or spend cap.
- Container/remote isolation parity for Codex (unchanged refusal).
- Routing deterministic nodes.

## Further Notes

- **Why the token price is not the lever.** 97% of Claude tokens are
  cache-reads, and Opus 5.5 cache-reads cost the same as Sonnet 5.5
  ($0.20/M, 0.05× input per the pricing page footnote). At constant tokens:
  - plan plus both reviews on Opus: about **+9%** API-equivalent;
  - build plus test on Haiku: about **−31%**, but turns grow superlinearly
    with cost (exponent 1.19, `adw-cost-02`), so extra turns from a weaker
    gear eat into that;
  - Fable per node: 1.6–2.0× Opus.

  That's why D6 puts the A/B first.
- **Where spend is, today:** build 42%, test 20%, plan 15%, review-fix 5%,
  build-fix 5%, repair 3%, review-spec 3%, review-standards 1%.
- **Long context:** Claude 4.6+ bills the full 1M context at standard rates,
  so `adw-cost-06`'s "200k premium" rationale no longer applies to Claude.
  Its hop remains a turn and context-size lever.
- **Ledger drift to fix alongside:** `adw-cost-02` is `done` with R3/R3a
  unchecked. Phase 0d discharges them.
- **The operator's global routing note** (`~/.claude/ai_docs/model-routing.md`)
  still names Fable 5 and is dated 2026-07-10, past its own 2026-10 retest
  trigger. Refresh it when Phase 0b records the Fable probe.
- **Sources (read 2026-10-03):**
  [Claude pricing](https://platform.claude.com/docs/en/about-claude/pricing),
  [Opus 5.5](https://www.anthropic.com/claude-opus-5-5),
  [Sonnet 5.5](https://www.anthropic.com/claude-sonnet-5-5),
  [Haiku 4.5](https://platform.claude.com/docs/en/models/haiku-4-5/overview).
  For the Codex catalog, the factory host's `~/.codex/models_cache.json` is
  the source of truth.
