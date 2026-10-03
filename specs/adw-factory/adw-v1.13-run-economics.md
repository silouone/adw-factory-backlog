# Amendment v1.13 — run economics: stop paying for work that is thrown away

## 1. Trigger — why this is an amendment, not a silent fix

Three things in this amendment contradict documents already in `specs/`, so under
`CLAUDE.md`'s amendment rule they cannot be changed in code alone:

1. **`adw-v1.8-agent-profiles.md` §3 binds `standard` to `medium` effort.**
   The shipped registry pins it to `high`, documented in-source as a deliberate
   divergence: `adw-profile-01`'s R4 required an absent-config run to emit
   options byte-identical to the pre-ticket `CHORE_MODEL="sonnet"` /
   `CLAUDE_REASONING_EFFORT="high"`, and §4 resolves level 5 to *literally the
   `standard` profile*. The spec table and the code have disagreed since
   2026-09-18 and the gap was flagged, not closed.
2. **`adw-v1.md` §5 makes `green | blocked` the terminal-outcome contract**, with
   no partial success. §4 of this amendment introduces a third shape — a run
   that reaches `green` while carrying unresolved advisory findings.
3. **`specs/constitution.md` Article V states breaching a ceiling has exactly
   one outcome: stop, mark blocked, hand the human a complete transcript.**
   S4/D4 gives the review↻fix loop's ceiling a second outcome — `next`, when
   every gate the loop re-checks in-process is already green — so Article V
   itself is amended alongside `adw-v1.md` §5, not silently diverged from in
   code. The retry stays bounded (the loop still stops at the same declared
   ceiling); only the terminal action at that ceiling gains a second case.

Everything else here is policy and tooling that the existing contracts already
permit.

## 2. Problem Statement

> *"the adw factory is SO SLOW it's hardly usable for me right now"* — operator,
> 2026-09-18, after a 110-minute run for work he does by hand in 30–45 minutes.

Measured across 50 banked runs and the 25 runs over 40 minutes
(`ai_docs/2026-09-18-why-runs-are-slow.md`, reproducible via
`scripts/run-scorecard.ts`):

| | |
|---|---|
| model think+generate vs tool execution | **80% / 20%** |
| full integration suite, run by the agent mid-stage | **144 calls, 268.2m — 80% of ALL tool time**, mean 5.8/run |
| review stages | 284.2m — **19.6% of model-stage time** |
| long runs ending `blocked` | **13 of 25 — 996m, 56% of long-run wall clock, producing nothing** |
| effort in use | **`high` on every stage of every run** |

The factory is not slow at writing code — reading and editing cost 2.4 minutes
across 3,387 calls. It is slow because it **runs four model stages at maximum
reasoning effort where the operator runs one**, re-verifies with a 112-second
command the engine is going to run anyway, and **throws away more than half of
its long runs at the last gate**.

Two specific traps compound this:

- **The operator has no middle gear.** The only way to lower effort today is
  `cheap`, which also drops `sonnet` → `haiku`. For `test` or `review-spec`
  that is too blunt to risk, so every stage stays at `high`.
- **A prompt cannot hold an economic constraint.** `prompts/feature-build.md`
  already tells the agent to use `bun run test:unit` (~21s), explains that
  `gates` re-runs the real suite regardless, and quantifies both. `adw-bug-06`
  cut the behaviour from ~15 calls/run to 5.8 — and it stopped there. The agent
  re-derives "I should check my work" every turn across 400 turns, and the
  cheapest honest reading of that is `bun test`.

## 3. Solution

Four changes, ordered by cost to adopt. The first two are policy the engine
already supports; the last two are small, well-seamed code.

**S1 — `review: false` on hard-gated tickets (policy, zero code).** A ticket
whose acceptance is fully expressed as machine gates opts out of the agent
reviewer. Already wired end-to-end and journalled as an explicit opt-out.

**S2 — a `swift` profile (sonnet + medium) and per-stage assignment (policy +
4 lines).** Gives the operator a middle gear so `agents: {test: swift}` is a
real option. `standard` stays `sonnet`/`high` and stays the level-5 default, so
no existing run changes behaviour.

**S3 — the full suite becomes unreachable inside an agent stage (one new
seam).** A deciding `PreToolUse` hook denies a full-suite invocation and returns
text naming the cheap command. The constraint moves out of prose and into the
engine.

**S4 — a non-converged review opens an annotated PR instead of blocking.** When
the review↻fix loop exhausts its rounds, gates are green *by definition* — the
work is shippable. Push it, open the PR, and render the unresolved findings into
the PR body, where Article IV already puts a human.

## 4. User Stories

1. As an operator, I want a ticket whose acceptance is fully machine-checkable to skip the agent reviewer, so that I stop paying 19.6% of model time for a second opinion my gates already give me.
2. As an operator, I want `review: false` recorded in the journal as an explicit opt-out, so that a skipped review is never indistinguishable from a review that silently failed to run.
3. As an operator, I want written guidance on when a ticket qualifies as hard-gated, so that the opt-out is a judgement I apply consistently rather than a habit I drift into.
4. As an operator, I want a profile that lowers reasoning effort without also dropping the model tier, so that I can economise on a stage without risking its quality.
5. As an operator, I want to assign that profile per stage, so that `test` and the review axes can run cheaper than `build` in the same run.
6. As an operator, I want adding the new profile to change nothing about runs that do not name it, so that adopting it is risk-free.
7. As an operator, I want the spec's profile table and the shipped registry to agree, so that reading the spec tells me what the factory actually does.
8. As an operator, I want the factory default to remain `standard`, so that the level-5 resolution rule in v1.8 §4 stays true.
9. As a build agent, I want the full integration suite refused during my stage with a message naming the cheap alternative, so that I learn the constraint instead of silently burning 112 seconds.
10. As a build agent, I want the refusal to tell me the `gates` node re-runs the full suite after my turn, so that I understand my own run was never the verification of record.
11. As a build agent, I want targeted test runs (`bun test <file>`) to stay available, so that I can still verify the code I just changed.
12. As a build agent, I want `bun run test:unit` to stay available, so that the refusal names an option I can actually take.
13. As an operator, I want the refusal to appear in the run transcript, so that I can see how often the agent tried and whether the message is working.
14. As an operator, I want the `gates` node itself to be unaffected by the agent-stage guard, so that the definition of done still runs the real suite. Amended 2026-10-02 (adw-gates-08, decision D-test-1): for a target that declares `paths` on a gate, the definition of done is "every gate that applies to the change" — a path-scoped gate whose paths the run never touched is recorded as skipped (v1.1 lanes, `Gate.paths`). `just verify` (lint, tsc, the full `bun run test`) remains the operator's whole-suite check before merging.
15. As an operator, I want a run whose review loop exhausted to produce a pull request rather than nothing, so that 76 minutes of gate-green work is not discarded.
16. As an operator, I want the unresolved findings rendered in that PR's body, so that I review the same material the agent could not resolve.
17. As an operator, I want unresolved findings visually distinct from findings a fix round resolved, so that I know which ones still need me.
18. As an operator, I want a run that exhausts its review loop to be clearly marked as such, so that an annotated PR is never mistaken for a clean one.
19. As an operator, I want gates-repair exhaustion to keep ending `blocked`, so that genuinely broken code still cannot reach a PR.
20. As an operator, I want CI-repair exhaustion to keep its current behaviour, so that this amendment changes exactly one terminal path.
21. As an operator, I want a scorecard that reports wall, model, tool, suite calls, review minutes and outcome per run, so that I can tell whether any of this worked.
22. As an operator, I want the scorecard to read only the sessions a run's own journal attributes to it, so that a neighbouring session's work is never reported as this run's.
23. As an operator, I want a recorded pre-change baseline, so that the comparison is against measurement rather than memory.
24. As an operator, I want each change adoptable independently, so that a regression in one does not force me to revert the others.
25. As a builder of the factory, I want the suite-refusal decision to be a pure function of the command string, so that its whole behaviour is testable without an SDK, a workspace or a subprocess.
26. As a builder of the factory, I want the deciding hook composed onto the existing opaque hooks bag, so that the agent node still neither constructs nor inspects hooks.
27. As a builder of the factory, I want the profile registry to stay a leaf module with no imports from `intake/` or `targets/`, so that the existing no-cycle invariant holds.

## 5. Implementation Decisions

### D1 — `review: false` ships no code (S1)

The field is parsed, validated, threaded to `LaneDeps.reviewEnabled`, and
consumed where the lane splices the review pair. The only deliverable is
**written policy** in `tickets/README.md`: a ticket qualifies as hard-gated when
every acceptance criterion is expressed as a gate command or a machine-checkable
assertion in its Verify block. Judgement-shaped work (naming, API shape,
architecture) does not qualify.

### D2 — `swift` is additive; `standard` and the default are untouched (S2)

The registry gains a fourth name bound to `(claude, sonnet, medium)`.
`standard` remains `(claude, sonnet, high)` and remains
`FACTORY_DEFAULT_PROFILE`, so v1.8 §4's level-5 rule is unchanged and an
absent-config run stays byte-identical.

**v1.8 §3's table is amended in two places:** add the `swift` row, and correct
`standard`'s effort column to `high` to match the shipped registry and the
byte-identity requirement that forced it. The divergence is closed by correcting
the table, not the code.

Both validators (`intake/ticket.ts` for `agent:`/`agents:` frontmatter,
`targets/loader.ts` for target config) resolve names through the registry's
existing predicate, so they accept the new name without modification. The
profile module stays a leaf.

### D3 — one new pure seam decides, an existing seam composes (S3)

A new **pure classifier** takes a Bash command string and returns either
*allow* or *deny with operator-facing text*. It is a leaf: no I/O, no clock, no
SDK types.

Classification:

| command shape | decision |
|---|---|
| `bun test` / `bun run test` with no path argument | **deny** |
| `bun test <path>` / `bun run test <path>` | allow |
| `bun run test:unit` | allow |
| anything else | allow |

The deny text names `bun run test:unit` (439 tests, ~21s) and
`bun test <file>`, and states that the `gates` node re-runs the full suite after
the turn ends regardless.

**Composition uses the existing precedent, not a new mechanism.** The agent node
already receives an opaque hooks bag it neither constructs nor inspects, and
`makeStreamWatchdog` already *wraps* that bag to add behaviour. The deciding
hook composes the same way: wrap the injected bag, add a `PreToolUse` matcher,
leave every other event forwarding untouched. cLens capture continues to observe
every call including the denied ones.

**Scope:** agent stages only. The `gates` node does not route Bash through the
agent and is unaffected. `red-check` and `baseline` are likewise unaffected.

### D4 — review exhaustion returns `next`, not `fail` (S4)

Today the review↻fix loop's round-exhaustion path calls `fail()`, which the
engine turns into `blocked`. It instead returns `next`, patching `ctx.data` with
the unresolved findings and a flag marking the run review-exhausted. The lane
then proceeds to its normal `commit` → `push` → `open-pr` tail.

**`open-pr` already renders review findings** — the verdict section, and the
`current` versus `fixed` distinction across rounds, shipped with `adw-m9-05`
(v1.3 R6). This change reuses that renderer and adds one section for unresolved
findings, plus a line in the PR body stating the review loop exhausted its
rounds. No new rendering machinery.

**Exactly one terminal path changes.** Gates-repair exhaustion, CI-repair
exhaustion, watchdog trips, hard stops, turn and wall-clock ceilings all keep
their current behaviour. A gate failure still blocks, because the code is
genuinely broken; a non-converged review does not, because gates are green by
definition at that point.

**Amends `adw-v1.md` §5:** `green` now means "the PR is open", which may include
a PR carrying unresolved advisory findings. It does not mean "a reviewer
approved it". The journal distinguishes the two, and the PR body states it
plainly.

**Amends `specs/constitution.md` Article V:** a ceiling breach's "exactly one
outcome" gains a second case, scoped to this one loop — see §1's third trigger
above and Article V's own amendment note.

### D5 — measurement is part of the deliverable

`scripts/run-scorecard.ts` is the instrument, already committed. It reports per
run: wall, model, tool, full-suite calls and minutes, review minutes, turns,
stage redraws, effort and outcome — with cLens sessions filtered to those the
run's own journal attributes to it.

**Baseline at 50 banked runs, recorded 2026-09-18:**

```
wall 2100.0m · model 1340.3m · tool 334.1m
full suite 144 calls / 268.2m · review 284.2m
green 18/50 (36%) · blocked wall 1154.7m
```

> **Baseline corrected 2026-09-19, after `adw-perf-04` shipped.** The figures
> above over-counted: the classifier used to write them matched `bun run
> test:unit` (its `test:` prefix satisfies a naive `\btest\b`) and
> path-scoped runs (`bun test test/foo.test.ts`) as full-suite calls. Both are
> the *cheap* path. `scripts/run-scorecard.ts` now splits on shell operators
> and discounts redirects and flags. Corrected, across 53 banked runs:
>
> | | inflated | corrected |
> |---|---:|---:|
> | full-suite calls | 144 | **55** |
> | minutes | 268.2m | **258.5m** |
> | seconds per call | 112s | **282s** |
> | mean calls per run | 5.8 | **1.8** (of the 30 runs that ran it at all) |
>
> The direction of §2's argument is unchanged and the per-call cost is **worse**
> than stated — a true full suite is ~4.7 minutes. But the frequency was
> roughly a third of what was claimed, so S3's recoverable total is smaller
> than §3's estimate. Quote this table, not the one above it.

**First post-hook measurement.** `adw-review-05` ran after `adw-perf-04` merged.
It attempted `bun run lint && bunx tsc --noEmit && bun test`; the guard
**denied** it (a `PreToolUse` with no `PostToolUse` in the capture), the agent
fell back to `bun run test:unit` at 16.7s, and the run finished **green in 17.4m
with 0 full-suite calls and 0.6m of total tool time**. The mechanism works.

Targets: suite calls/run **1.8 → 0**; review minutes on an opted-out ticket
**→ 0**; blocked share of long-run wall clock **56% → materially lower**.

## 6. Testing Decisions

A good test here asserts **external behaviour at the highest seam available**
and would survive a rewrite of everything beneath it. Article I binds: the red
test comes first.

- **The suite classifier** is the ideal unit under test — a pure
  string → decision function. Table-driven cases covering each row of D3's
  table, plus the shapes seen in the banked captures (`bun test 2>&1 | tail -20`,
  `bun run lint && bunx tsc --noEmit && bun test`, `bun test test/intake/select.test.ts`).
  A compound command containing a bare full-suite invocation must deny.
  Prior art: `test/pipeline/retry-policy.test.ts`, the codebase's existing
  pure-classifier test.
- **Hook composition** asserts that wrapping preserves every other event's
  forwarding — the property `makeStreamWatchdog`'s own tests already establish
  for the same bag. Prior art: `test/pipeline/nodes/agents.test.ts` and the
  watchdog's identity assertions.
- **The profile registry** has an existing test asserting the exact sorted key
  set, which goes red on the new name before any other edit — Article I for
  free. Extend it to assert `swift`'s triple, that `standard` is unchanged, and
  that `FACTORY_DEFAULT_PROFILE` still resolves to `standard`. Prior art:
  `test/pipeline/profiles.test.ts`.
- **Frontmatter and target validation** for the new name go through the existing
  parse tests. Prior art: `test/intake/ticket.test.ts`, `test/targets/loader.test.ts`.
- **Review exhaustion** is tested at the fix-loop seam: given a verdict that
  never converges, the loop returns `next` carrying unresolved findings rather
  than `fail`. Prior art: `test/pipeline/review/fix-loop.test.ts`, which already
  covers the round-exhaustion path and gained a reproducing test in `adw-bug-13`.
- **PR body rendering** extends the existing snapshot. Prior art:
  `test/pipeline/nodes/__snapshots__/open-pr.test.ts.snap`.
- **No test asserts a wall-clock duration.** Speed is measured by the scorecard
  against the recorded baseline, not asserted in the suite — a timing assertion
  would be the flakiest test in the repo.

## 7. Out of Scope

- **Removing or folding the `test` stage** (250.9m, 17% of model-stage time).
  The feat lane is specified with three stages; changing that is its own
  amendment.
- **Bounding `plan` redraws** (30 executions across 25 runs, 153.2m). Needs
  investigation into *why* `plan` retries before a fix can be proposed.
- **Defect A of `adw-bug-13`** — review rounds as independent redraws rather
  than iterations. `adw-bug-13` fixed Defect B (flake tolerance) only. S4 makes
  non-convergence survivable; it does not make the loop converge.
- **Capture attribution** — `runs/<id>/.clens/sessions/` holds foreign sessions
  (70 of 169 across banked runs). `run-scorecard.ts` filters defensively, but
  the web UI's metrics rail still reads them. Correctness, not economics; its
  own ticket.
- **Per-stage provider** (`adw-profile-02`). `swift` is Claude-path only.
- **Changing any model slug.** The registry's pinned literals are unchanged
  apart from the new row.
- **Blanket-lowering the factory default.** `standard` stays `high`; this
  amendment gives the operator a gear, it does not shift into it.

## 8. Further Notes

**Why the prompt was not simply rewritten again.** `prompts/feature-build.md`
already carries a correct, specific, quantified instruction not to run the full
suite, including the reason it is pointless. `adw-bug-06` shipped against this
same behaviour and halved it. The residue is not an information gap — the agent
re-derives "verify my work" every turn, and across 400 turns the constraint
decays. **An economic constraint expressed as a prompt sentence decays; the same
constraint expressed as a refusal does not.** That generalises beyond this
amendment.

**Why S4 is the largest number and the smallest diff.** 996 minutes of long-run
wall clock produced nothing, and in every one of those runs the gates were
green. The machinery to render findings into a PR body already exists. The
change is a routing decision — `fail` becomes `next` — not new capability.

**Adoption order.** S1 and S2 are policy and can be applied to the next ticket
dispatched. S3 and S4 are one ticket each. Run `scripts/run-scorecard.ts` after
each to compare against §5 D5's baseline.
