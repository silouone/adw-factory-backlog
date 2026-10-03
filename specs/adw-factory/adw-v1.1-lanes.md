# Amendment v1.1 — three purpose-built build lanes (`chore | bug | feat`)

> **Status:** proposed 2026-07-22, **refined 2026-07-23** (two operator grilling
> sessions — decisions locked below), **reshaped 2026-07-23** to align with the
> operator's source vision (`ai_docs/2026-07-23-source-vision-factory-map.md`).
> **Approved 2026-07-23** by the operator (Silou) — amendments included; the 9
> refined child tickets blessed as-is; red tests validator-gated for 01–07, the
> operator gating the live bars (08/09).
> **Amends:** `adw-v1.md` §2 (non-goals); `adw-v1-plan.md` decisions 2 & 6 and
> §5 (ticket contract). **Binding:** `constitution.md` — unchanged. Plan §2 Gate
> III (2 agent node types) is **NOT** amended: the new `plan`/`test` stages
> **reuse the `build` node** (see below), so no new agent node *type* is added.

## Trigger (why this is allowed, not a silent diverge)

Constitution Art. II: *"One lane until it earns trust."* The chore lane has
earned it — six real cLens chores merged (clens-001…006, incl. E2B PR #17),
meeting §1's gate ("prove every piece of shared plumbing before any lane
multiplication") and the ≥3-merged NFR. `adw-v1.md` §2's "bug, feature, and
hotfix lanes" non-goal is **un-deferred for `bug` and `feat`**. Hotfix, a router
agent, and the "Your ADW" extension slot stay deferred.

One commitment in `adw-v1-plan.md` is amended, deliberately and in writing:

- **Decision 6 — routing.** Type still selects a graph by a **code switch**, not
  a router agent. Three graphs behind a switch is not the dynamic "Factory
  Router Agent" of the source vision — that (and "Your ADW") remains deferred.

**Gate III is NOT amended.** Aligning `bug`/`feat` with the source vision adds
`plan`, `build` (tests-first), and `test` *stages* — but mechanically each is a
**fresh agent query capturing a summary**, i.e. an **instance of the existing
`build` node** (parameterized by prompt + output key + name), exactly as `bug`'s
test-only and fix phases are. The second and only other agent node type stays
`repair-resume` (a session-resuming query). So there remain **two agent node
types**, and reusing one node across stages is *more* Art. VIII-compliant ("one
representation"), not less. Art. III is satisfied by the *prompts*: `plan`
decomposes intent into a consumed artifact, `test` authors coverage — judgment a
function can't do — while **running** tests/lint/typecheck stays deterministic in
the gates node.

## Problem Statement

*From the operator's perspective.* The factory serves one kind of work order —
a `chore`. My real backlog is bugs and features, and those are not mechanical:
a **feature** deserves the *most complete* workflow in the house — it should be
**planned**, built test-first, and have its **test coverage hardened** before it
ever reaches me. A **bug** must ship with a regression test the machine *watched*
go red before the fix — a green suite at the end proves nothing. And the reason
this factory exists — the **moat** — is that it runs end-to-end in an isolated
**E2B** sandbox; a new lane is only "done" once it has produced a merged PR *on
E2B*, not just locally. My earlier draft made `feat` a bare prompt-swap on the
chore graph; against my source map that is naive, and I want it fixed.

## Solution

*From the operator's perspective.* Three ticket types, **three purpose-built
graphs**, selected by a code switch on `type` (no router agent):

- **`chore`** — the existing, proven graph, unchanged.
- **`feat`** — a **Planner-led** pipeline: an agent decomposes the referenced
  spec into a plan, a build agent implements **tests-first** against that plan,
  a test agent **expands coverage / edge cases**, then the deterministic gates
  prove green. The richest ADW in the house.
- **`bug`** — a **Plan-led, machine-enforced red-first** pipeline: base proven
  clean → an agent plans the reproduction → an agent writes **only** a failing
  test → a deterministic **red-check** proves it goes cleanly red → the same
  agent (resumed) fixes → gates prove green. A dirty base or a non-clean red
  **blocks** with a precise reason — no PR built on a spoofed proof.

Isolation (worktree / container / E2B) stays orthogonal: every type runs on
every kind. Work is proven cheap on **worktree** first, but v1.1 is not *done*
until `bug` **and** `feat` each land a merged PR on **E2B** — the moat.

## The change — three node graphs

Selection is a **code switch on `ticket.type`** at the orchestrator (`dispatch`
already switches on `type`), picking one of three lane builders. The engine,
gates node, push/open-pr, sync-pr-state/ci-round, workspace kinds, observability,
auth, and the caps model are otherwise unchanged.

### Agent stages — still two node types

Two agent node *types* (Gate III intact); the graphs use several *instances* of
the `build` type, each parameterized by prompt + output key + node name (so each
still gets its own journal entry + span — Art. VI):

| Stage (node instance) | Node type | Judgment in the prompt (Art. III) |
|---|---|---|
| `plan` | `build` (fresh query → `ctx.data.plan`) | Decomposes the spec/bug into a **plan artifact** the build stage consumes. |
| `build` | `build` (fresh query → `agentSummary`) | Implements code under uncertainty (existing). `bug` uses it for the test-only phase too. |
| `test` | `build` (fresh query) | **Authors/expands** the feature's tests — coverage, edge cases. Does *not run* them. |
| `fix` (bug) | `repair-resume` shape (resuming query) | Implements the fix, resuming the test-only session. |
| `repair` | `repair-resume` (existing) | Fixes a gate failure in-session. |

**Running** tests, lint, and typecheck stay 100% deterministic in the gates node
— no agent was introduced where a function suffices (Art. III). No new agent node
*type* is added; `build` is reused, which is the Art. VIII-preferred "one
representation."

**The `fix` step** (ticket adw-gates-09). Where a target declares `fix`, the
gates node runs it as code before every gates pass that follows an agent node
(after build/test, and after each repair round) — shown as `fix → gates` in the
graphs below. It never runs before the baseline (the base is measured as-is) nor
on the post-rebase re-gate (no agent ran since the commit). Its exit code never
gates: a non-zero exit is journaled as a `fix-step` event and lint still
decides. It is not an agent because formatting is a function (Art. III):
`biome check --write` is deterministic and idempotent.

### Graph A — `chore` (unchanged)

```
dispatch → provision → assemble-prompt → build → fix → gates(↻repair≤3)
  → commit → push → open-pr
```

Byte-identical to today — same prompt (`chore-build.md`), caps, and node list;
regression-covered by the existing determinism/snapshot suite.

### Graph B — `feat` (Planner-led, tests-first + coverage hardening)

```
dispatch → provision
  → assemble(plan)  → PLAN agent      (spec → plan artifact)
  → assemble(build) → BUILD agent     (tests-first, consumes {{plan}})
  → assemble(test)  → TEST agent      (expand coverage / edge cases)
  → fix → gates(↻repair≤3) → commit → push → open-pr
```

- **Tests-first is preserved** (operator decision): the build agent writes tests
  then implements to green; the test agent *afterward* hardens coverage — it does
  not author tests to fit already-written code.
- **The map's "Test → fail → back" loop is the deterministic `gates ↻ repair`
  loop**, not a separate agent-judged loop: after the test agent runs, the gates
  (incl. the new tests) are the failure signal; a red gate routes to `repair`
  (Art. III — the machine check owns the loop). `feat` therefore has **one**
  loop (gates↻repair); no per-node override needed.

### Graph C — `bug` (Plan-led, two-phase machine-enforced red-first)

```
dispatch → provision → 🟢 base-green-check
  → assemble(plan)      → PLAN agent          (bug → repro-strategy artifact)
  → assemble(test-only) → BUILD agent         (write ONLY a failing test, consumes {{plan}})
  → 🔴 red-check (↻ revise ≤2 → build test-only)
  → assemble(fix)       → BUILD agent (RESUME) (implement the minimal fix)
  → fix → gates(↻repair≤3) → commit → push → open-pr
```

> **As of the 2026-09-14 amendment (end of document): `provision → 🟢
> base-green-check` above reads `provision → baseline → baseline-green-check`
> in the shipped graph** — `baseline` (a node that predates this whole
> document) already runs the full gate suite on the untouched checkout right
> after `provision`; `base-green-check` used to re-run it a second time.

1. **base-green-check** (deterministic, new). Run the target's **full gate suite
   on the untouched checkout**; all green → proceed; any red → **block** ("target
   base not in a provable-clean state"). Ordered **before** the plan agent — do
   not spend agent tokens planning on a dirty base. Art. VII guarantees nothing
   is *pushed* on a red result, but says nothing about the base a run is *cut
   from*, and a live target's `main` can be red. **Superseded 2026-09-14 — see
   the amendment at the end of this document: `base-green-check` no longer
   exists; the bug lane now wires `baseline-green-check` in this graph
   position instead.**
2. **plan agent.** Analyzes the bug, emits a repro-strategy plan artifact
   consumed by the test-only phase.
3. **build (test-only).** Writes **only a failing reproducing test**, no fix.
4. **red-check** (deterministic, new) — the one **new pure seam**. Runs a
   **run-all gate execution that collects every result** (NOT the stop-at-first
   `gates` node — see Impl. decision 7) and classifies:
   `classifyRed(orderedGateResults, testGateName) → "clean-red" | "test-passed"
   | "broken-test"`. **clean-red** = the `test` gate fails AND every non-test
   (well-formedness) gate passes, on the proven-clean base. Non-clean → a
   **bounded revise loop (cap 2)** back to the test-only build with the precise
   reason; exhaustion → **block**.
5. **build (fix, RESUME).** Resumes the test-only session (mirrors `repair`,
   S2.2) and implements the minimal fix.
6. **gates(↻repair≤3)** onward — the existing tail; the fix must leave every
   gate, including the now-passing reproducing test, green.

### The designated `test` gate — convention + fail-fast (bug only)

The base-green-check and red-check need to know which gate is **behavioral** (the
red signal) vs **well-formedness** (lint/typecheck, must stay green at red-check).

- **Convention:** the gate named `test` is behavioral; every other gate is
  well-formedness. cLens conforms (`lint`, `typecheck`, `test`) — zero config.
- **Fail-fast (Art. IX):** a `bug` ticket whose target has **no gate named
  `test`**, or where `test` is the **only** gate, is **rejected before
  provisioning**. No `testGate:` config field added until a second target proves
  the convention insufficient (Art. II).

### Plan-artifact threading

The `plan` node returns its artifact on `ctx.data` (e.g. `plan`); the subsequent
`assemble(build)` renders it into a **new `{{plan}}` placeholder** the build
templates carry. `assemblePrompt`'s signature is unchanged; `KNOWN_PLACEHOLDERS`
gains `plan` (present only in plan-consuming templates). chore never uses it.

### Engine — per-node retry override (backward-compatible)

The engine's retry mechanism has a single target (`lane.repair`) and one
`maxRounds`. `bug` needs two independent loops — red-check↺test-only (cap 2) and
gates↺repair (cap 3) — so a node may declare an optional
`retry?: { target, maxRounds }`; on a `retry` result the engine uses the node's
override, else falls back to `lane.repair`/`lane.maxRounds`. The round counter
**stays in the engine** ("the round counter lives here, never in nodes"),
preserving per-phase journal + span granularity (Art. VI) and declared ceilings
(Art. V). chore/feat declare no override → byte-identical loop behavior.

## User Stories

*Operator:*

1. As the operator, I want to dispatch `bug` and `feat` tickets, so the factory
   fixes defects and builds features, not just chores.
2. As the operator, I want a **feature planned before it is built**, so a
   non-trivial feature is decomposed by an agent rather than improvised.
3. As the operator, I want the feature built **tests-first** and then have its
   **coverage hardened** by a test agent, so features arrive well-tested, not
   just compiling.
4. As the operator, I want a bug fix to ship with a regression test the machine
   **watched go red** before the fix, so the test provably reproduces the bug.
5. As the operator, I want a bug **planned** first, so the reproduction strategy
   is reasoned about before a test is written.
6. As the operator, I want a bug run to **block** on a dirty base or a non-clean
   red (after a bounded revise), so no PR is built on a spoofed proof.
7. As the operator, I want the red-check to tell the agent *exactly why* its test
   was not a clean red, so a cheap miss is fixed in-run, not blocked.
8. As the operator, I want `chore` byte-identical, so generalizing cannot regress
   the one thing already earning trust.
9. As the operator, I want per-ticket context (feature spec pointer, bug repro)
   in the **ticket body**, so no new frontmatter field is needed.
10. As the operator, I want a `bug` to a target without a separable `test` gate
    **rejected loudly** before provisioning.
11. As the operator, I want every type to run on every isolation kind unchanged.
12. As the operator, I want the new machinery shaken down on **worktree** first,
    cheap and observable.
13. As the operator, I want v1.1 **not done** until `bug` and `feat` each land a
    merged PR on **E2B** — the moat.
14. As the operator, I want graph selection to stay a **code switch** (no router
    agent), so the system stays predictable (decision 6).
15. As the operator, I want the plan and test agents to earn their keep — a
    **plan artifact the build consumes**, a **test agent that authors coverage**
    — so I never pay an agent where a function would do (Art. III).

*Build agents:*

16. As the plan agent, I want to emit a concrete plan the build stage consumes,
    so my judgment shapes the implementation.
17. As the feat build agent, I want the plan and a tests-first instruction, so I
    implement against a decomposition, not a vague prompt.
18. As the feat test agent, I want the built feature and its tests, so I add the
    coverage and edge cases the build missed.
19. As the bug build agent (phase 1), I want to write **only** a reproducing
    test, so the red-check has a clean red to verify.
20. As the bug build agent (phase 2), I want my phase-1 session resumed with the
    test confirmed red, so I implement the minimal fix with full context.

*Builder team:*

21. As a builder, I want each new graph composed at the **`LaneSpec` seam** (a
    lane builder returning an ordered node list), so I add graphs without
    touching the engine's core loop.
22. As a builder, I want the red-check decision in **one pure classifier**, so I
    test every branch with fabricated inputs and zero I/O.
23. As a builder, I want the engine's per-node retry override to be
    backward-compatible, so chore/feat behavior is proven byte-identical.
24. As a builder, I want `plan`/`test` agent nodes to mirror the existing `build`
    node idiom (injected `AgentQuery`, faked in tests), so no new seam appears.

## Implementation Decisions

1. **Three node graphs, selected by a code switch on `ticket.type`** (decision 6
   preserved — a switch picking a *graph* is not a router *agent*). One lane
   builder per type; the orchestrator seeds the right templates.
2. **`Ticket.type` union → `"chore" | "bug" | "feat"`**; lane registry accepts
   the three; any other type rejected (S1.3). Only ticket-contract change.
3. **Still two agent node types** (Gate III intact). The `plan`/`build`/`test`
   stages are **instances of the reusable `build` node** (parameterized by
   prompt + output key + name); `fix`/`repair` are the `repair-resume` type. The
   judgment lives in the prompts (Art. III); running tests stays deterministic in
   gates. Reuse over a new type is Art. VIII-preferred.
4. **Per-ticket context is body-only** — no new frontmatter field. Feature spec
   pointer / bug repro live in the body (→ `{{ticketBody}}`); referenced spec
   files are opened in-workspace (pointer, not inlined). Not hard-required — a
   missing repro yields a natural block via the revise loop.
5. **Prompt templates (committed, versioned — N5):** `feature-plan.md`,
   `feature-build.md`, `feature-test.md`, `bug-plan.md`, `bug-build-test.md`,
   `bug-build-fix.md` (+ unchanged `chore-build.md`). Plan-consuming build
   templates carry a new `{{plan}}` placeholder. Selection is a code switch by
   type (and, for bug, by phase).
6. **Plan-artifact threading** via `ctx.data` → `{{plan}}`; `assemblePrompt`
   signature unchanged, `KNOWN_PLACEHOLDERS` gains `plan`.
7. **red-check runs a run-all gate execution collecting every result** — NOT the
   stop-at-first-fail `gates` node (S2.1), which would truncate before a `test`
   gate that is not last and let the classifier emit a false `clean-red`.
   base-green-check may reuse the standard stop-at-first execution (the first red
   is enough to block).
8. **`classifyRed` pure classifier** (the one new pure seam):
   `(orderedGateResults, testGateName) → "clean-red" | "test-passed" |
   "broken-test"`; `clean-red` iff the `test` result is a failure and every
   non-test result is a pass.
9. **base-green-check + red-check deterministic nodes**; both over the existing
   gate-execution seam; errors carry ticket id + node name (Art. IX).
10. **Engine per-node retry override** (`retry?: { target, maxRounds }`),
    backward-compatible; used by `bug`'s red-check (→ test-only, cap 2). The
    counter stays in the engine.
11. **Fix phase resumes the test-only session** (mirrors `repair`, S2.2).
12. **Bounded revise loop, cap 2** (new declared ceiling — Art. V) on non-clean
    red; no phase-1 test-only path guard (the red-check self-polices a leaked
    full fix; a partial leak still leaves a genuine red).
13. **Fail-fast precondition** for the bug lane: no separable `test` gate →
    reject before provisioning (Art. IX).
14. **`plan`/`test`/`build` agent nodes share the injected `AgentQuery` seam**;
    `plan` and `test` mirror the `build` node idiom (a fake in tests). No new
    boundary.
15. **Caps: all three types reuse chore defaults** (`sonnet`, 120 min / 400
    turns) to start. Turns are per-agent-invocation (each stage gets a full
    budget); **run-wide minutes** is the pressure point — `feat` (plan+build+
    test) and `bug` (plan+test+fix) run more stages under one wall-clock. Tune
    via per-ticket `caps:` if a worktree run bites; do not guess (decision 12).

## Testing Decisions

Assert **external behavior at a seam**, never internal wiring. Prior art is
abundant; mirror it.

- **`classifyRed` (new, pure):** all three branches with fabricated ordered
  gate-result arrays, zero I/O. Prior art: `test/pipeline/nodes/gates.test.ts`,
  `test/intake/ticket.test.ts`.
- **base-green-check / red-check nodes:** faked `workspace.exec` (green vs a red
  gate); assert `next` vs `block`, and the run-all collection (red-check must run
  *all* gates even when an earlier one fails).
- **`plan` / `test` agent nodes:** faked `AgentQuery`; assert the plan node's
  artifact lands on `ctx.data` and the build prompt renders `{{plan}}`; assert
  the test node runs after build and before gates. Prior art: the build/repair
  node tests.
- **engine per-node retry override:** chore/feat (no override) behave
  byte-identically; a node with an override routes retries to its declared target
  and enforces its own ceiling. Prior art: `test/pipeline/engine.test.ts`.
- **three lane compositions + flows:** `featLane` runs plan→build→test→gates;
  `bugLane` gates the fix on `clean-red`, drives the revise loop and blocks after
  2, resumes the fix with the phase-1 `sessionId`, blocks on a red base;
  type→lane selection; fail-fast rejection.
- **template selection + `{{plan}}` render**; the chore assembled-prompt
  **snapshot stays byte-identical** (machine guarantee of chore identity).
- **ticket contract:** three types parse; a fourth rejected (S1.3).
- **live bar (operator-gated, the hard gate):** offline-green never suffices (the
  M7 lesson). The staged acceptance below is the live bar.

## Out of Scope (unchanged non-goals + newly explicit)

- **Hotfix lane, a Factory Router Agent, the "Your ADW" extension slot, and the
  mid-flow Approve/Reject human gate** the source vision draws — all deferred.
  Routing stays a code switch over the three registered types.
- **A distinct machine "test-fail" loop for `feat`** — the deterministic
  `gates ↻ repair` loop is the failure signal (Art. III); no agent-judged feat
  loop.
- **Phase-1 "test-only" path guard** and per-target test-path globs (decision 12).
- **An explicit `testGate:` target-config field** — convention stands (decision
  13).
- **Per-type caps/model tuning** — all reuse chore defaults; per-ticket overrides
  remain the escape hatch (decision 15).
- **Proving the reproducing test targets the *specific* bug** — the machine
  proves a well-formed test went red on a clean base and green after; that it
  reproduces *the* bug is semantic (prompt + review).

## Acceptance (staged — E2B is the moat)

**Stage 1 — worktree shakedown (build phase).** A real `bug` travels Graph C
(base-green → plan → test-only → clean red-check → fix → gates) to a **merged**
PR on **worktree**, **and** a `feat` travels Graph B (plan → build tests-first →
test coverage → gates) to a **merged** PR on **worktree**. Proves the new
machinery where it is cheapest and most observable.

**Stage 2 — E2B (the mandatory done bar).** `bug` **and** `feat` each travel to a
**merged** PR on the **E2B remote** kind — the Planner-led feature pipeline and
the two-phase red-check both running inside a remote sandbox. **v1.1 plan+build
is not done until both new types have run to a merged PR on E2B.** (chore on E2B
is already proven — PR #17.)

Plus, always:

- `chore` **byte-identical** (regression-covered by the existing snapshot suite).
- `bun run lint && bunx tsc --noEmit && bun test` green, incl. new red-first
  tests: `classifyRed`'s three branches, base-green block, the run-all
  collection, the revise-loop cap, the engine retry override (+ chore/feat
  byte-identity), plan-artifact threading, `test`-node ordering, template
  selection, the widened type contract, and the fail-fast precondition.

## Residual Risks (named, with backstop)

- **`feat`'s test agent could author tests that pass trivially** rather than
  meaningfully hardening coverage. Backstop: operator review at merge; not
  machine-enforced (coverage-quality gating is speculative until a run shows the
  need — Art. II).
- **A `gates ↻ repair` round could weaken/delete the bug's reproducing test** to
  force green. Backstop: operator review at merge; not machine-enforced (Art. II).
- **Clean-red is structural, not semantic** — proves a well-formed test went red
  then green, not that it reproduces *the* bug. Prompt + review close the gap.
- **More agent stages pressure the run-wide minutes budget.** Backstop:
  per-ticket `caps:` override; tune from journal data (decision 15).

## Surgical code deltas (the whole scope)

1. `Ticket.type` union → `"chore" | "bug" | "feat"`; lane registry accepts three.
2. Six committed prompt templates (feature ×3, bug ×3) + a new `{{plan}}`
   placeholder in the plan-consuming build templates; template selection switch.
3. Parameterize the `build` node (node name + output key) so `plan`/`build`/
   `test` are instances of it (no new node type) + plan-artifact threading
   through `ctx.data`.
4. Engine: an optional per-node `retry?: { target, maxRounds }` override
   (backward-compatible; chore/feat unaffected).
5. A pure `classifyRed` + a run-all `red-check` node + a `base-green-check` node.
6. `featLane()` and `bugLane()` `LaneSpec` builders; orchestrator selects among
   `chore`/`feat`/`bug` lanes by `ticket.type`.
7. The bug-lane fail-fast precondition (no separable `test` gate → reject).

## Amendment 2026-09-14 — `base-green-check` answered from `ctx.data.baseline`

Ticket `adw-gates-02-stop-paying-for-the-suite-four-times`. This document
predates `baseline`/`baseline-green-check` entirely (they landed later, in
`adw-auto-01-baseline-gate-snapshot` and the 2026-09-13 operator decision) —
by the time those two nodes existed, the bug lane already had its own
`base-green-check` (decision 9, above), and the two were **deliberately not
unified**: `bug.ts`'s own header comment argued they "answer a DIFFERENT
question" — `baseline` records *what's already known-bad*; `base-green-check`
proves *the cut base is provably clean right now*.

**That reasoning predates measurement and does not survive it.** Both nodes
run `target.gates` on the identical untouched checkout, back to back, at the
same sha, in the same run. Profiling `adw-depends-enforce`
(`tickets/adw-gates-02-stop-paying-for-the-suite-four-times.md`'s own
evidence) measured **9 min in `baseline` + 7 min in `base-green-check` — 16
minutes duplicating an already-known answer**, before a single agent token was
spent. "Provably clean right now" and "what's already known-bad" are answered
by the SAME gate exec; running it twice does not make the second answer any
more current — the base tree has not moved between the two.

**Decision:** `bug.ts` now wires `makeBaselineGreenCheckNode()`
(`nodes/baseline.ts`) — already used by `chore`/`feat`, PURE over
`ctx.data.baseline`, zero gate execs of its own — in `base-green-check`'s old
graph position, right after `baseline`. `base-green-check`
(`nodes/red-check.ts`) is deleted, not left behind (Surgical Changes: remove
what your change orphans); its only callers were `bug.ts` and its own test.

**Two behavioral deltas this decision knowingly accepts:**

1. **Wording (strict improvement).** The block message changes from
   `"target base not in a provable-clean state"` to
   `baseline-green-check`'s `"gate \"X\" is already failing on the base commit
   <sha> … refusing before any agent token is spent (S2.7)"` — it now names
   the sha and, for the `test` gate, the failing test names. Strictly more
   informative, not a regression.
2. **Config-error timing (the real regression, accepted).** A fresh
   `base-green-check` re-run caught a misconfigured gate command (exit 127)
   immediately, zero agent tokens spent — `red-check.ts`'s
   `base-green-check` ran the standard stop-at-first execution and any
   non-zero exit blocked. After this merge, that same misconfiguration makes
   `baseline`'s own `runAllGates` throw, producing `ok:false`; since
   `baseline-green-check` only ever inspects `ok:true` results (Art. VI: a
   baseline that could not run must never itself block a run), it
   **advances**. The config error is not silently swallowed: `red-check` has
   no E3 config-error guard of its own, so it treats the still-broken gate as
   an ordinary gate failure, which can perturb `classifyRed` (a spurious
   `test-passed`/`broken-test` classification, potentially spinning the
   bounded revise loop) before the run finally terminates — still `blocked`,
   never a false `green` — either when revise exhausts its cap or, if the
   revise loop happens not to trigger, at the terminal `gates` node (which
   *does* keep its E3 guard). Net effect: the SAME misconfiguration is still
   caught before push in every case; it is simply caught **later** — after
   plan/test-only tokens are spent — than the old dedicated re-run allowed.
   **Accepted deliberately**: adding a discriminated field to
   `BaselineResult` to distinguish "config error" from "any other unrunnable
   cause" is structure invented for a case with no observed occurrence
   (Art. II — a capability earns its way in when a run demonstrates the
   need). Test case:
   `test/pipeline/lanes/bug.test.ts` — *"baseline ok:false (config-error-shaped):
   baseline-green-check advances, the run proceeds, and still terminates
   blocked — never a false green."*

**A middle ground was considered and rejected**: re-running the base-green
check only on a cache HIT (trusting a cache miss's own fresh run, only
duplicating when the cheap answer might be stale). Rejected — it undermines
exactly the savings this amendment exists to capture for the lane paying the
most (bug: 4 pipeline-level full-suite execs → 3), and adds a branch nothing
in the evidence asked for.

**The cache-poisoning question is out of scope, not closed by this merge.** A
concern going in: a cached `ok:true` baseline measured under a degraded
environment (Docker down) could be trusted by a later run with Docker back up.
Noting explicitly: `base-green-check`'s fresh re-run never actually closed
this gap either — it just re-executes the gate commands and would see the
SAME degraded exit code (a docker-down `bun test` exits 0, per
`adw-gates-01`'s own finding) and advance identically. This is
`adw-gates-01`'s territory (cache completeness), not this ticket's, and is
**not** a reason to keep the duplicate re-run.

**Graph C, updated for this amendment** (decisions 1 and 9 above, and the
ASCII diagram, are left as originally written — historical record of what was
built and why, per this repo's own "never silently drop stale reasoning" norm
— this amendment is the correction of record):

```
dispatch → provision → baseline → baseline-green-check
  → assemble(plan)      → PLAN agent          (bug → repro-strategy artifact)
  → assemble(test-only) → BUILD agent         (write ONLY a failing test, consumes {{plan}})
  → 🔴 red-check (↻ revise ≤2 → build test-only)
  → assemble(fix)       → BUILD agent (RESUME) (implement the minimal fix)
  → gates(↻repair≤3) → commit → push → open-pr
```

`baseline-green-check`'s own decision record and tests live in
`nodes/baseline.ts` / `test/pipeline/nodes/baseline.test.ts` (it predates this
amendment — `chore`/`feat` already used it). This amendment only changes which
lane wires it where.

No change to: the gates node's own stop-at-first logic (reused for base-green;
red-check uses a separate run-all execution), push/open-pr, sync-pr-state/
ci-round, workspace kinds, observability, auth, or the target-config contract.

## Amendment 2026-09-16 — `red-check` narrows the `test` gate to the added test

Ticket `adw-perf-03-red-check-runs-the-whole-suite-to-watch-one-test-go-red`.
Measured on `adw-bug-09-…`: `red-check` took **3.1 minutes** to answer a
single yes/no question — did the one test `build-test-only` just wrote go
red — by running the target's full `test` gate, **1832 tests across 72
files**. The same shape as the adw-gates-02 amendment above: a full-suite
gate exec paying for an answer a narrower execution already carries.

**The two concerns `red-check.ts`'s header comment conflates.** The run-all
requirement ("runs ALL gates UNCONDITIONALLY, collecting every result even
past a failure") is about *gate ordering* — never stop before reaching a
`test` gate that is not last, or `classifyRed` sees a truncated result set
and can emit a false `clean-red`. It is not an argument that the `test`
gate's own *command* must exercise the entire suite. This amendment narrows
the command; the run-all ordering guarantee is untouched.

**Decision:** `Gate` gains an optional `scopedCmd` field — a command
template containing a `{{files}}` placeholder. When the gate named `test`
declares one, and `git status --porcelain --no-renames` in the worktree
shows at least one changed path matching the test-file naming convention
`bun test` itself already enforces (`.test.`, `.spec.`, `_test_`, or
`_spec_` in the filename), `red-check` renders `scopedCmd` against exactly
those paths and execs that instead of the gate's full `cmd`. Every other
gate — lint, typecheck, any well-formedness gate — always execs its full,
unscoped `cmd`, unchanged. A target that never declares `scopedCmd` sees
**no behavioral change at all**: `red-check` never issues a `git status`
call for it and runs every gate's `cmd` exactly as before.

No baseline-sha threading was needed to find "the file(s) the turn added":
the worktree's `HEAD` already *is* the base commit at `red-check` time —
`dispatch` commits land on the target repo's own ticket file, not the
worktree; `plan` is read-only; neither `build-test-only` nor
`revise-test-only` ever commits. Uncommitted `git status` output is exactly
the turn's delta.

**Behavioral delta accepted — a correctness fix, not just a speed one.**
Today, a `test` gate failure caused by an *unrelated* full-suite test (a
flaky or already-red one that slipped past `baseline`'s own N=2
reproduction check) is indistinguishable, at whole-gate granularity, from
the added test's own red: `classifyRed` sees `test: fail` either way and
reports `clean-red` — a false proof, since the machine never actually
observed the added test itself go red. Narrowing the `test` gate's
execution to only the added file(s) removes this: an unrelated failure
elsewhere can no longer contaminate the added test's own pass/fail signal.

**What stays exactly as documented above, unchanged by this amendment:**
`classifyRed`'s signature and three-way classification; the run-all
collect-every-result guarantee; the node's name, graph position, and
`retry` wiring; the `gates` node's own full-suite, stop-at-first-INTRODUCED
run (still the verification of record before push); `baseline.ts`'s own
`runAllGates` (still needs the real, whole-tree result as the
pre-existing-failure subtraction basis for `gates.ts`).

`scopedCmd` is opt-in per target, per gate — declared today only for
`adw-factory` (and, optionally, `clens`; both share the identical
`bun run test` toolchain). No other committed target (`sabado.json`'s
`just test`, the Docker-compose-gated targets) declares it: no
target-appropriate scoped-invocation syntax was verified for those
toolchains, and extending the capability to one nobody has measured a
payoff for yet is exactly the speculative generality Art. II rules out.

### `Gate.paths` — a gate that only applies to the changes that touch it (2026-10-02, adw-gates-08, decision D-test-1)

**Why.** Four adw-factory test files drive live infrastructure — real Docker
(`cli.container`, `container.contract`, `container.reattach`) and billed E2B
sandboxes (`e2b.contract`) — about 200 s and a known flake source per gate pass,
paid at the baseline, after the build, after every repair round and after a
rebase. A flake there can block a run for a change that never touched a
container. (`e2b.test.ts` fakes `E2bOps`, is not live, and stays in `test`.)

**Decision.** `Gate` gains an optional `paths: readonly string[]` — glob
patterns relative to the repo root, validated like `scopedCmd` (non-empty array
of non-empty strings; anything else is a load error naming `gates[i].paths`).
A gate without `paths` behaves byte-identically to before; no other committed
target declares it.

- **`gates` node.** A gate with `paths` runs only if the run's changes touch at
  least one matching path. "Changes" = committed changes against the merge-base
  with `base` (`git diff --name-only base...HEAD`) ∪ uncommitted ∪ untracked
  files (`git status --porcelain -uall`; gates run before the `commit` node),
  with `.adw/` excluded. If the changed set cannot be determined (no base, a git
  error) the gate **runs** — fail closed, never a silent skip.
- **Skipped, never dropped (Art. V).** An untouched gate is recorded as
  `{ outcome: "pass", skipped: true, reason: "paths [...] untouched by this run" }`
  in the journal's `gates` rider and the PR gate summary. A skipped gate counts
  as not-failing everywhere `outcome` is read; it is never executed (no
  `exec`, no report clearing).
- **Re-evaluated every pass.** Repair, ci-round, rebase and review-fix rounds
  each run the `gates` node afresh, so a repair that starts touching a matching
  path brings the gate in on that round.
- **`baseline` node.** There is no diff at the baseline, so a `paths` gate is
  never snapshotted: it is recorded as skipped in the baseline record (the cache
  shape stays explicit; no schema bump — an older record merely lacks the gate,
  which the `gates` node already treats as introduced). A run that does trigger
  the gate therefore has **no baseline for it, and any failure is attributed to
  the run**. That is the honest default for a gate that only runs when the run
  touched its code.
- **`red-check`.** Its `scopedCmd` narrowing is unchanged (the `test`-named gate
  only), but a `paths` gate gets the same skip decision as in `gates`;
  otherwise the bug lane would run the live gate on every run.
- **adw-factory.** `test` ignores the four live files
  (`--path-ignore-patterns`, repeated once per file); `test-live` runs exactly
  those files, writes its own `.adw/report-live.xml`, and is gated on
  `src/workspace/container*`, `src/workspace/e2b*`, `src/workspace/types.ts`,
  `containers/**` and the live test files and gates.

**Accepted risk.** A change outside `paths` (for example in `src/cli.ts` or a
lane) can break the container lane without the gate seeing it. Two mitigations:
the faked container/e2b unit tests still run in `test`, and `just verify` (lint,
tsc, the full `bun run test`) is the operator's whole-suite check before
merging.

## Amendment 2026-10-03 — a failed pass still reports every gate (adw-gates-11, decision D1)

**Why.** S2.1 stopped the `gates` node at the first introduced failure, so
repair saw one gate's failure per round and found the next only by burning
another. hq-07 failed all four passes at `lint` with 118 `tsc` errors never
shown; hq-05's rounds swung between lint and tsc. The 3-round budget could not
cover work nobody could see.

**Decision.** S2.1 becomes "route on the first introduced failure; report every
gate". After it, the remaining gates still run, **report-only**:

- They are recorded in `gates` with `afterFailure: true`; their introduced
  failures ride the repair payload as `alsoFailing` (each with its own
  `REPORT_TAIL_CHARS` share, so one noisy gate cannot push out another) and are
  rendered by `prompts/repair.md`. The first failure stays the headline.
- They never change routing: the run goes to repair because of the headline
  failure, exactly as before. A base-red headline still fails at once, before
  any later gate runs, and a later gate that cannot run (exit 126/127) is
  recorded as failing, never turned into a `fail`.
- A later failure is classified against the baseline as usual: pre-existing
  rides `gates` only; introduced rides `alsoFailing`.
- The repair loop's livelock fingerprint also covers the `alsoFailing` tails,
  else a round that only shrinks a later gate's errors would read as stuck.

**D1 — cost of a report-only test gate: option (a).** adw-factory's `test`
gate runs for minutes, and adw-gates-02 exists to stop paying for the suite
repeatedly. Every remaining gate runs report-only **except** one declaring
`reportOnlyAfterFailure: false`; it is recorded
`{ outcome: "pass", skipped: true, afterFailure: true, reason }`, never dropped
silently (Art. V). adw-factory's `test` declares it, so a lint or typecheck
failure does not buy a suite run (the cost gates-02 removed); silou-hq, clens
and sabado declare nothing and report every gate. Rejected: (b) run only the
gates declared before `test` — it hides test failures behind a lint failure;
(c) a target-level switch — coarser than the per-gate cost it models.

**Callers with no repair after them** (`ci-round`, `rebase-round`, review
`fix-loop`, `gates-rebased`) construct the node with `reportAfterFailure:
false` and keep stop-at-first: extra suites there buy nothing.
