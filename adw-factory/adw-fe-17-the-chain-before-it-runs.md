---
id: adw-fe-17-the-chain-before-it-runs
type: feat
status: done
priority: 2
created: 2026-09-15
depends: [adw-fe-16-the-run-screen-does-not-move]
attempts: [{"runId":"adw-fe-17-the-chain-before-it-runs-1789515239428","branch":"adw/adw-fe-17-the-chain-before-it-runs-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-17-the-chain-before-it-runs-1789515239428/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/57","provider":"claude","model":"sonnet"}]
---
# Show the whole lane chain from the first second, not one node at a time

A lane is an **ordered, statically known list of nodes** — that is the whole
premise of this repo. The run screen throws that away and draws only what has
already happened, so a run that is 30 seconds old looks like a run with one
node, and there is no way to see how much is left.

Proposed by the operator 2026-09-15: *"since we know beforehand what steps it
will take, we can preload and pre-render them — empty at first."*

The proposal is right. Two things must be settled before it can be built, and
neither is a coding decision.

## Blocker 1 — the journal cannot name its own lane early

`deriveLane` (`projection.ts:113-122`) infers the lane from nodes **already
seen**: `build-test-only` → bug, `plan` → feat. At `dispatch` and `provision`
time — precisely when the ghost chain is most useful — the journal cannot say
which of the three chains is running.

`run-start` (`journal.ts:157-165`) carries `isolation` and an optional
`target`. It does **not** carry the ticket's `type:`.

Two candidate fixes, and this ticket must pick one before any rendering work:

- **(a) Widen the journal.** Add `lane` (or `type`) to `run-start`, optional
  so existing journals still read — the exact precedent `target` set in
  `adw-fe-01`. Cheapest, and it makes the lane a recorded fact rather than an
  inference. `deriveLane` keeps its inference as the fallback for old runs.
- **(b) Read the ticket store from `src/web/`.** No web module reads
  `tickets/` today. This would also unlock `depends:` visibility, which
  `adw-v1.5` §2 lists and nothing delivers — but it couples the web layer to
  the ticket store, and `adw-v1.4` proposes moving that store elsewhere.

**Recommendation: (a).** It is smaller, it is reversible, and it does not
bet against `adw-v1.4`.

## Blocker 2 — where does a node with no duration go on a time axis?

The x-axis is **time**. A node that has not run has no start, no end and no
duration. Three options, each with a cost:

- **(i) A separate "up next" strip**, outside the time axis, listing the
  remaining chain in order with no time claim. Honest; costs vertical space
  and does not show *where* the run is in its arc.
- **(ii) Reserve span from an estimate** — median duration per node from the
  banked journals. Shows the arc; **fabricates a number**, and the axis jumps
  whenever reality exceeds the estimate. This repo is emphatic about the
  opposite (an unknown model yields no estimate; `cap-none` vs captured-empty
  are deliberately distinct).
- **(iii) Ghost slots at equal width** after the live node, explicitly marked
  not-to-scale. Shows what is left; puts two scales on one axis.

**This is a spec question, not an implementation detail.** Per the amendment
rule it is decided in `specs/adw-v1.2-live-view.md` (or a v1.6) before it is
built. The author's lean is **(i)**: it adds the missing information without
putting a fabricated duration on an axis whose entire credibility is that its
numbers are measured.

## Then, and only then

- [ ] Pending nodes render from the first frame, visibly empty — no glyph, no
      duration, no purpose text claiming work that has not happened.
- [ ] A pending node is unmistakably distinct from a deterministic one: `k-det`
      is already dashed and dim, and "free" must not read as "not yet run".
- [ ] A node that never runs (a bounded loop that exits early, a lane that
      blocks before its tail) resolves to a **named** terminal state, never a
      ghost left hanging forever.
- [ ] Retries — `gates ↻ repair` and `red-check ↻ revise` — are not in the
      static chain. State what a ghost chain shows for a loop it cannot
      predict the length of.

## Verify

- [ ] A run screen opened 5 seconds into a `feat` run shows all of
      `plan → build → test` plus the shared head and tail.
- [ ] No fabricated duration appears anywhere, on any axis, for a node that
      has not run.
- [ ] A blocked run's un-run tail is named as un-run, not left ghosted.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Live refresh itself (`adw-fe-16`, which this depends on — a ghost chain on a
static page is worth little). `depends:` graph visibility, unless blocker 1
is resolved as (b), in which case say so and scope it separately.
