---
id: adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane
type: bug
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane-1789468802650","branch":"adw/adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane","workspace":"/Users/silouane/adw-factory/runs/adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane-1789468802650/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane-1789552021123","branch":"adw/adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane-1789552021123/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# A *running* agent node is drawn on the `code` lane and labelled "deterministic · no tokens" — the row is only correct once the node is over

> Observed 2026-09-15 on `adw-perf-01-the-feat-lane-pays-for-the-same-context-three-times-1789465948398`,
> while `plan` was live. The screen the operator opens **to watch a run** puts
> the run's only working agent on the deterministic lane, and says it costs
> nothing.

## 1. What happened

Measured off that run's own journal, mid-flight, on the `code` lane:

| node | left% | width% |
|---|---|---|
| baseline | 0.12 | 24.71 |
| baseline-green-check | 24.84 | 0.00 |
| assemble-plan | 24.84 | 0.00 |
| **plan** (live) | **24.84** | **69.86** |

`plan` — a **feat-lane agent node**, 12m29s of paid model work in progress —
is on the shared `code` row. The rail beside it therefore reads:

```
⌘ code
deterministic · no tokens
share 100%
```

Every clause of that is false for the thing actually running. And because the
agent block is folded onto the workspace row, it also collides with the
zero-duration markers parked at the same instant (`baseline-green-check`,
`assemble-plan` at 24.84% with `min-width:26px`), which is what the operator
first reported as a rendering overlap. **The overlap is a symptom; the lane
assignment is the cause.**

The same run, re-read after `plan` finished, puts it on a `k-agent` lane of
its own with its model — correct. So the row is right only in hindsight, which
is exactly when nobody is watching.

## 2. Root cause

`gantt.ts:138-144`:

```ts
const pushBlock = (block, kind: NodeEndDetails["kind"] | undefined) => {
  const role = kind === "agent" ? block.node : WORKSPACE_LANE;
  laneFor(role).push(block);
};
```

`kind` is read from `event.details?.kind`, and `details` exists **only on
`node-end`** (`journal.ts` `NodeEndDetails`). An in-progress node has no
`node-end`, so `kind` is `undefined`, so `role` falls to `WORKSPACE_LANE`.

This is already named in the module header as a known limitation, with the
proper fix identified — *"a journal-schema change (carrying kind on
`node-start` too)"* — and deferred. This ticket is that fix. Nothing else in
`tickets/` covers it: `adw-fe-17` is about nodes that have **not started**
(ghost chain); this is about a node that **has** started being drawn on the
wrong row.

## 3. Two candidate fixes — pick one before writing rendering code

- **(a) Widen the journal: carry `kind` on `node-start`.** The header's own
  named proper fix. Makes the lane a **recorded fact** at the moment the node
  begins, not an inference. Optional field, so every banked journal still
  reads — the exact precedent `target` set on `run-start` in `adw-fe-01`. The
  engine already knows the node's kind when it starts it.
- **(b) A static node-name → kind table in the projection.** The agent-node
  names are a small, closed, lane-independent set — `plan`, `build`, `test`,
  `build-test-only`, `build-fix`, `revise-test-only`, `repair`, the `review-*`
  stages — and `render-run.ts`'s own `PURPOSE` map is already exactly this
  shape. No schema change. But it duplicates knowledge that lives in
  `src/pipeline/lanes/*.ts`, and silently mis-rows any node renamed or added
  later — the drift this repo keeps refusing elsewhere.

**Recommendation: (a), with (b) as the read-side fallback for journals banked
before the field existed** — mirroring how `deriveLane` keeps its inference
for old runs. That keeps every banked journal renderable without making a
guess the load-bearing path.

## 4. The rail must stop lying too

Fixing the row is not enough on its own:

- `deterministic · no tokens` is printed for any lane whose role is
  `WORKSPACE_LANE`. Once a live agent gets its own lane this resolves, but the
  lane's `model` is read from the **last block carrying `usage`** — and a
  live agent node has no usage yet. It must read `model unknown` or the
  attempt's resolved model, never blank and never a wrong claim.
- `share` is computed per lane against the run span; with one lane it reads
  `100%`. It must recompute correctly once the agent lane exists mid-run.

## 5. Red test first (Art. I)

1. A journal whose last event is a `node-start` for `plan` (no `node-end`)
   puts that block on a **`plan`** lane, not `workspace`. Red today.
2. That live agent lane renders as `k-agent`, and its rail does **not** say
   `deterministic · no tokens`.
3. A live **deterministic** node (`provision`, `commit`) still lands on
   `workspace` — the fix must not send every unfinished node to its own lane.
4. A journal banked **before** `node-start` carried `kind` renders
   byte-identically to today (the fallback path).
5. `share` across lanes sums correctly while one node is still in flight.

## Verify

- `bun run lint && bunx tsc --noEmit && bun test` green.
- A run screen opened while `plan` is running shows `plan` on its own lane
  with a non-deterministic rail.
- Re-reading the same journal after the run finishes yields the same lane
  assignment it showed live — the row never moves between renders.

## Out of scope

Ghost/pending nodes that have not started (`adw-fe-17`). The
`min-width:26px` marker stacking for zero-duration nodes — a real, separate
geometry defect, worth its own ticket; this one only stops the agent block
being piled onto the same row.

---

## Resolution — 2026-09-17, fixed by hand

Fix **(a)**, as recommended in §3, with the read-side fallback intact.

**Producer.** `EngineNode.journalKind?: NodeEndDetails["kind"]` — the node
declares the kind its own `node-end` will carry, and `runNode` stamps it on
`node-start` (omitted when undeclared). Declared `"agent"` on the six real
agent-usage node factories (`build`, `repair`, `ci-repair`, `review-fix`,
`revise-test-only`, `build-fix`) and `"review"` on `review`, which carries
`spanType: "AGENT"` but journals `kind: "review"` — §3's trap, and the reason
`spanType` is not the discriminator.

**Reader.** `buildGanttView` carries the declared kind on its `PendingNode`
and prefers it at **both** ends of the bracket, not only on the end-of-stream
flush. That last part is what discharges the Verify line "the row never moves
between renders": an agent node can end with **no** `usage` (an early
`fail()` carries none), so a details-only read would file the *finished*
block back on `workspace` after the live read had already put it on its own
lane. A journal banked before the field existed declares nothing, falls
through to `details.kind`, and renders byte-identically.

**Rail (§4).** `kindOf` (`render-run.ts`) and `kindOfBlock` (`card.ts` — the
board mini-chart had the same defect) consult the lane role as a last resort,
after `usage`/`metrics`/`gates`. A block in a non-`workspace` lane is an
agent block by `buildGanttView`'s own invariant. `model unknown` is left as
the honest rail for a live agent that has reported no usage yet; `share`
needed no code change — it recomputes correctly once the lane exists.

**Verified.** `bun run lint && bunx tsc --noEmit && bun test` — 2175 pass, 4
skip, 0 fail. Plus an end-to-end walk through the real engine and a real
journal file (a hanging agent node): `node-start` carries `kind:"agent"` on
disk, `plan` lands on its own lane while live, renders `k-agent`, and its
rail reads `model unknown` rather than `deterministic · no tokens`.

Out-of-scope items untouched as declared: `adw-fe-17`'s ghost chain and
`adw-fe-19`'s `min-width:26px` marker stacking (the reported overlap goes
away as a side effect of the block no longer sharing the workspace row, but
no geometry was changed).
