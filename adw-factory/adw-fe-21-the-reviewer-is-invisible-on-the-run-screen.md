---
id: adw-fe-21-the-reviewer-is-invisible-on-the-run-screen
type: feat
status: done
priority: 1
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-fe-21-the-reviewer-is-invisible-on-the-run-screen-1789677026767","branch":"adw/adw-fe-21-the-reviewer-is-invisible-on-the-run-screen","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-21-the-reviewer-is-invisible-on-the-run-screen-1789677026767/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# An agent that hits an LLM, burns tokens and can block the run renders as a nameless box on the "no tokens" lane

> The run screen has one job: show what the run is doing and what it costs.
> `adw-m9-06` added two agent stages to every lane, and the Gantt files both of
> them under `code · deterministic · no tokens` — the lane reserved for work
> that makes no model call at all.

## Evidence

Run `adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run-1789673356704`,
observed on the run screen:

- Lanes rendered: `code` (deterministic · no tokens), `plan`
  (claude-sonnet-5), `build-test-only` (claude-sonnet-5), `build-fix`
  (claude-sonnet-5).
- `review-standards` **ran, made a model call, and failed the run** — and has
  **no lane**. It appears only as a small unlabeled box with a red ✗ inside the
  `code` lane, indistinguishable from a deterministic wrapper node.
- The `UP NEXT` strip *does* list `review-spec` — so `src/web/chain.ts` already
  knows the review stages exist. Only the Gantt's lane assignment does not.

## Root cause — one equality test

`src/web/gantt.ts:152`:

```ts
const role = kind === "agent" ? block.node : WORKSPACE_LANE;
```

`src/pipeline/nodes/review.ts:228` declares `journalKind: "review"` — while
declaring `spanType: "AGENT"`, so the *tracer* already treats it as an agent.
`"review" !== "agent"`, so every review node falls through to the shared
`workspace` lane.

The file's own header states the intended contract, which the code no longer
honours:

> *"Lane assignment keys off `node-end.details.kind`, never a node-name
> allowlist: any agent-kind node lands in its own lane, named after itself —
> for agent nodes the node name IS the role. Every other node (deterministic or
> gate-kind, including `baseline`) shares one fixed `workspace` lane."*

`review` is neither deterministic nor gate-kind. It is an agent node by every
property that matters — it has a model, a session, a token cost and a
watchdog — and it is being laned as though it were neither.

**The durable fix is to invert the test.** Today's `=== "agent"` is an
allowlist of one: every future agent-ish kind regresses this bug silently and
identically. The workspace lane should be chosen by what genuinely belongs
there (deterministic and gate kinds), so an unrecognized kind fails toward
*visibility*, not toward the "no tokens" lane.

## Why it matters beyond cosmetics

- **Cost is unattributable.** Each agent lane carries its model and its share
  of the run; review carries neither. The run billed 17,629 tokens with two
  review stages' worth of spend folded into a lane captioned *no tokens*.
- **M9's own exit metric depends on this.** `specs/adw-v1.3-review-lane.md` §6
  decides whether the reviewer was worth building by measuring what it costs
  against the operator-review minutes it saves. That measurement cannot be made
  from a screen that does not show the reviewer.
- **`adw-profile-01` makes it worse.** Once stages can run different models,
  a per-lane model label is the primary way to see which profile each stage
  resolved to. A review stage with no lane shows no model.
- **It is the exact defect `adw-fe-18` already fixed once**, for a different
  cause: *"the screen an operator opens to watch a run put the run's one
  working agent on the deterministic lane."* That ticket fixed *when* the kind
  is read (node-start vs node-end). This one is *which* kinds count.

## Requirements

- [ ] `review-standards`, `review-spec` and `review-fix` each render in their
      OWN lane, named after the node, exactly as `plan`/`build`/`test` do.
- [ ] Lane assignment is decided by what belongs in the workspace lane
      (deterministic + gate kinds), NOT by an equality test against `"agent"`,
      so a future kind cannot regress this the same way.
- [ ] Each review lane shows its resolved model, like every other agent lane.
- [ ] Review token usage is attributed to its own lane and is no longer folded
      into a lane captioned *deterministic · no tokens*. If the caption is
      derived rather than literal, it must stop claiming "no tokens" for a lane
      that has them.
- [ ] `adw-fe-18`'s fix is preserved: the lane is decided from `node-start`'s
      declared kind and never moves between a live read and a post-hoc read of
      the same journal. A review node in flight must already be on its own
      lane, not arrive there when it ends.
- [ ] A journal banked before `journalKind` existed still renders
      byte-identically (the existing compat contract in `gantt.ts`'s header).
- [ ] `review-fix` is a retry TARGET, never a `nodes` member — it must lane
      correctly when it runs without being added to `DISPLAY_CHAINS`
      (`src/web/chain.ts` deliberately omits it, like `repair`/
      `revise-test-only`).

## Verify

- [ ] Red tests first, against the real projection: a journal containing
      `review-standards`/`review-spec` records produces one lane PER review
      node, each carrying its model — RED today (they collapse into
      `WORKSPACE_LANE`).
- [ ] A review node with only a `node-start` (in flight) is already on its own
      lane — the `adw-fe-18` property, now for review.
- [ ] A review node that FAILS renders in its own lane with its failure, not as
      an anonymous box on the workspace lane — the observed bug.
- [ ] Regression: deterministic and gate nodes (`baseline`, `gates`,
      `red-check`, `commit`, `push`, `open-pr`, `assemble-*`) all still share
      `WORKSPACE_LANE`, and a pre-`journalKind` journal renders unchanged.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] Re-render the evidence run above (`runs/adw-bug-11-...-1789673356704/`)
      and confirm `review-standards` appears as its own lane with its model and
      its failure.

## Out of scope

Changing what the reviewer does, its prompts, or its verdict contract
(`adw-bug-12` owns the parse/retry/evidence defects). Adding `review-fix` to
`DISPLAY_CHAINS`. Any change to the up-next strip, which already lists the
review stages correctly.
