---
id: adw-m8
type: epic
status: done
priority: 3
created: 2026-07-23
depends: [adw-m5]
children:
  - adw-m8-01-ticket-type-union
  - adw-m8-02-engine-retry-override
  - adw-m8-03-plan-test-agent-nodes
  - adw-m8-04-prompt-templates
  - adw-m8-05-red-check-nodes
  - adw-m8-06-feat-lane
  - adw-m8-07-bug-lane
  - adw-m8-08-integration-worktree
  - adw-m8-09-e2b-moat
attempts: []
---
# EPIC M8 — Three purpose-built build lanes (`chore | bug | feat`)

The build lane generalizes to three ticket types, each a **purpose-built graph**
selected by a **code switch on `type`** (no router agent — decision 6). Aligned
to the operator's source vision (`ai_docs/2026-07-23-source-vision-factory-map.md`).

**Source of truth:** `specs/adw-v1.1-lanes.md` (proposed 2026-07-22, refined +
reshaped 2026-07-23). Read it, `constitution.md`, `adw-v1.md`, `adw-v1-plan.md`
before any child. On contradiction or gap: stop, propose a spec amendment.

## The three graphs

```
chore:  dispatch → provision → assemble → build → gates(↻repair≤3) → commit → push → open-pr

feat:   dispatch → provision → PLAN → BUILD(tests-first) → TEST(coverage)
          → gates(↻repair≤3) → commit → push → open-pr

bug:    dispatch → provision → 🟢base-green → PLAN → build(test-only)
          → 🔴red-check(↻revise≤2→test-only) → build(fix·resume)
          → gates(↻repair≤3) → commit → push → open-pr
```

**Gate III intact (2 agent node types).** The `plan`/`build`/`test` stages are
**instances of the reusable `build` node** (parameterized by prompt + output key
+ name); `fix`/`repair` are the `repair-resume` type. Judgment lives in the
prompts (Art. III); *running* tests/lint/typecheck stays deterministic in gates.
**clean red** = the `test` gate FAILS and every non-test gate PASSES on a
proven-clean base; test gate by convention (gate named `test`), fail-fast
otherwise.

## Children

| Ticket | Scope | Deps |
|--------|-------|------|
| 01 ticket-type-union | `Ticket.type` → `chore\|bug\|feat`; registry | m1-01 |
| 02 engine-retry-override | Optional per-node `retry?:{target,maxRounds}`; chore/feat byte-identical | m1-05 |
| 03 plan-test-agent-nodes | `plan` + `test` agent nodes; `{{plan}}` placeholder + plan threading | m1-07, m1-09 |
| 04 prompt-templates | 6 templates (feat ×3, bug ×3); selection; chore snapshot identical | 01, 03 |
| 05 red-check-nodes | pure `classifyRed` + run-all `red-check` + `base-green-check` | m1-08 |
| 06 feat-lane | `featLane()` plan→build→test→gates; orchestrator selection | 03, 04 |
| 07 bug-lane | `bugLane()` base-green→plan→test-only→red-check→fix; fail-fast | 02, 03, 04, 05 |
| 08 integration-worktree | Stage 1 live bar: bug + feat → merged PR on worktree | 06, 07 |
| 09 e2b-moat | Stage 2 live bar (MOAT): bug + feat → merged PR on E2B | 08 |

## Exit criteria

`chore` byte-identical (snapshot-covered). A `bug` travels Graph C and a `feat`
travels Graph B, each to a **merged** PR — first on **worktree** (Stage 1), then
on **E2B** (Stage 2). **The epic is not done until `bug` AND `feat` have each
landed a merged PR on E2B** — the moat. Full suite + lint + tsc green throughout.

## Progress (2026-07-23)

**Offline wiring complete — children 01–07 all `done`, validated + reviewed.**
Built by 4 parallel + 2 parallel builder agents, each with an independent
validator (red-genuineness verified empirically); the two integration lanes
(06 feat, 07 bug) wired into cli.ts by the orchestrator. Two-axis `/code-review`
(base 0ccfe17): Standards 0 hard / 6 judgement calls, Spec conformant (no missing
reqs, no scope creep, 3 documented deviations). Full suite **789 green**, lint +
tsc clean, chore snapshot **byte-identical**. Code-review dedups folded
(TEST_GATE_NAME one-representation; `fail` reuse). Follow-up `adw-m8-10`
(lane-wrapper consolidation) filed, non-blocking.

**Remaining — the operator-gated live bars (the hard gate; M7 lesson: offline
green is necessary but NOT sufficient):**
- `08` Stage 1 — a real cLens `bug` (Graph C) + `feat` (Graph B) each to a
  **merged** PR on **worktree**. Real tokens; the operator dispatches + merges
  (Art. IV — the factory is merge-incapable).
- `09` Stage 2 (MOAT) — the same two on **E2B**. The mandatory done bar; on green
  the epic closes.

## Epic DONE (2026-07-26)

All four live proofs **merged** to real cLens PRs: bug #18 + feat #19 (worktree,
Stage 1), bug #20 + feat #21 (**E2B, the moat**). Both new lanes (`bug` Graph C
with machine-enforced clean-red, `feat` Graph B Planner-led) traveled to a merged
PR inside a remote E2B sandbox — the mandatory done bar is met. `chore` stayed
byte-identical throughout. Two E2B live findings (heavy-feat vs the ~60-min
sandbox ceiling; a transient E2B transport stream-stall caught by the liveness
watchdog) were remote-infra, not lane/code bugs — recorded in `adw-m8-09`.
Follow-up `adw-m8-10` (lane-wrapper consolidation) remains queued, non-blocking.
