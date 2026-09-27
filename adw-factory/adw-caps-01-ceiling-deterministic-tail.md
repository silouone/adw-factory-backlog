---
id: adw-caps-01-ceiling-deterministic-tail
type: feat
status: done
priority: 1
created: 2026-09-12
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-caps-01-ceiling-deterministic-tail-1789203994866","branch":"adw/adw-caps-01-ceiling-deterministic-tail","workspace":"/Users/silouane/adw-factory/runs/adw-caps-01-ceiling-deterministic-tail-1789203994866/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/7","provider":"claude","model":"sonnet"}]
---
# A turn-ceiling breach must not abort the deterministic tail of a run

> Minted 2026-09-12 from the retrospective on run
> `adw-par-01-repo-commit-lock-1789164087662`: 84 minutes of correct,
> verified work that the factory threw away 4 nodes from the finish line.
> The work was salvaged by hand as PR #6. It should not have needed to be.

## The retrospective

Reconstructed from the journal, not from impressions.

| node | start@ | duration | turns | tokens |
|---|---|---|---|---|
| dispatch / provision / assemble-* | 0.0m | ~0 | — | — |
| `plan` | 0.0m | 17.0m | 56 | 65 042 |
| `build` | 17.1m | **43.9m** | **296** | 126 148 |
| `test` | 61.0m | 23.0m | 70 | 29 837 |
| | | **84.0m** | **422** (cap 400) | 221 027 |

**It is not a performance problem.** The run was clean: **zero `round` events,
zero retries, zero `watchdog` trips** — every node succeeded first time. Pacing
was steady (3.3 / 6.7 / 3.0 turns per minute). Token-per-turn is 426–1 161,
the signature of ordinary small tool round-trips, not thrash. Wall clock used
**84 of its 120-minute cap (70%)** — wall was never the binding constraint.

The run died on a **cumulative counter**, breached by **22 turns — 5.5%**.

### Cause 1 — the design defect (this ticket)

`engine.ts:303` applies the ceiling gate at **every node boundary**, blind to
what the next node *is*. When it fired, `test` had just ended
`outcome: "next"` and the remaining path was:

```
gates → commit → push → open-pr
```

**None of those consume a single agent turn.** They are deterministic code.
A turn ceiling exists to bound *agent* spend (Art. V) — it should refuse
further **agent** work, not prevent a run from *finishing* through
deterministic nodes it has already paid for.

Had it done so, this run would have opened PR #6 **by itself**. The difference
between an 84-minute blocked run and an 84-minute green run is this one rule.

**The rule cannot be "always allow the tail."** `repair` is an agent node and
is reachable from a red `gates`. So: on a turn breach, deterministic nodes may
proceed; the first *agent* node refuses and the run blocks. Green gates → PR;
red gates → blocked, correctly, because repair would need turns.

### Cause 2 — tuning (worth fixing, not the headline)

`FEAT_CAPS = { minutes: 120, turns: 400 }` (`feat.ts:116`) is a lane-wide
default calibrated on smaller work. This ticket was 18 files, +1 909/−239.
`caps` is already per-ticket overridable (`ticket.ts:80`) and this ticket did
not use it — an operator-side miss, not a code defect.

### Cause 3 — the stage that burst the budget returned nothing

`test` spent **70 turns / 23 minutes / 17% of the budget** and, by the agent's
own report, its hardening tests *"all passed against the existing
implementation"* with **no source change and zero defects found**. The tests
have regression value, so this is not waste — but it is the stage that pushed
352 → 422, and it found nothing. Worth its own investigation; **not in scope
here.**

## Requirements

- [ ] The engine's boundary gate distinguishes a **turn** breach from a
      **wall-clock** breach. A wall-clock breach still aborts everything —
      the run is out of time by definition. A turn breach must not.
- [ ] On a turn breach, the engine continues through **deterministic** nodes
      and refuses the first **agent** node it reaches, blocking there with a
      reason naming the ceiling.
- [ ] Nodes declare whether they consume agent turns. Deriving it from
      "did it return `usage`" is a post-hoc signal and useless *before*
      running the node — the lane must be able to answer it up front.
- [ ] A run that finishes its deterministic tail after a turn breach ends
      with its **real** outcome (green if gates pass and the PR opens), not
      `blocked`. The breach is recorded as a journal event either way, so the
      run's history still says the ceiling was hit.
- [ ] An external abort and the wall-clock deadline keep **exactly** today's
      behaviour. This ticket narrows one branch, not the Art. V guarantee.
- [ ] Raise `FEAT_CAPS.turns` — recommend **600**. State the rationale in the
      constant, as the existing caps do.

## Verify

- [ ] **The regression test for this whole ticket:** a lane whose turn count
      is already past the cap when it reaches a deterministic node runs that
      node, and opens its PR, rather than aborting.
- [ ] The same lane, on reaching an **agent** node past the cap, blocks with
      the ceiling reason and does **not** invoke the query seam.
- [ ] Turn-breach → red gates → `repair` is refused → run blocks. The green
      path must not become a way to skip repair.
- [ ] Wall-clock breach still aborts at the next boundary regardless of node
      kind — unchanged, asserted explicitly.
- [ ] External abort unchanged.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Amendment

> **[AMENDED] 2026-09-27**, ticket `adw-caps-03-a-mid-stream-breach-still-salvages`:
> this ticket's deterministic tail covered a turn ceiling breached at a node
> **boundary** — the engine's own cumulative accounting (`turnsUsed` vs
> `caps.turns`, checked between two nodes). After
> `adw-bug-23`/`adw-bug-25` changed what counts as a turn (a distinct API
> call, not the SDK's `num_turns`), a single agent node's own usage can no
> longer breach at that boundary: `build.ts`'s own mid-stream check
> (`turnsSeen > ctx.caps.turns`, inside its stream-reading loop) fires
> first, before the node ever returns to the engine, and returned an
> ordinary `fail()` — which this ticket's kind-aware boundary gate never
> saw, so the run blocked unconditionally and discarded whatever the node
> had already written to disk. `adw-caps-03` closes that gap: a node may
> now report a mid-stream ceiling breach as a distinct `NodeResult.fail`
> marker (`turnCeilingMidStream`), and the engine routes it through the
> IDENTICAL deterministic-tail machinery this ticket built — deterministic
> nodes still run, the first subsequent agent node still refuses, and the
> breach is still recorded. This ticket's own requirements and Verify
> checks are otherwise unchanged; they described the boundary case
> correctly, they just were not the whole story.

## Out of scope

- Why the `test` stage spends turns for zero defects (cause 3). Real, and a
  separate investigation.
- Per-node turn budgets. A cumulative cap is the right shape; this ticket
  fixes what it *does on breach*, not how it counts.
- Any change to the wall-clock ceiling's semantics.

## Run log

_(empty)_
