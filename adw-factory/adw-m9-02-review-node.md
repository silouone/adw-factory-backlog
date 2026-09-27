---
id: adw-m9-02-review-node
type: feat
status: done
priority: 1
created: 2026-09-12
epic: adw-m9
depends: [adw-m9-01-review-verdict-contract]
attempts: [{"runId":"adw-m9-02-review-node-1789345192390","branch":"adw/adw-m9-02-review-node","workspace":"/Users/silouane/adw-factory/runs/adw-m9-02-review-node-1789345192390/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/29","provider":"claude","model":"sonnet"}]
---
# The `review` agent node — read-only over a diff, write DENIED

## Context

`specs/adw-v1.3-review-lane.md` R1 + R4, and the Gate III amendment.

A reviewer cannot be an instance of `build`: `build` mutates the workspace and
returns a summary, while a reviewer must be read-only over a diff and return a
typed verdict. Making it a `build` instance would hand the reviewer write access
to the code it is judging — the exact conflict of interest the node exists to
remove. So this is the THIRD agent node type, and `adw-v1-plan.md` §2 Gate III
goes 2 → 3.

**R4 is the load-bearing requirement: write-denial must be ENFORCED, not
requested.** A prompt saying "do not edit" is not a control. The run must be
unable to write.

## Deliverables

`src/pipeline/nodes/review.ts` (a `makeReviewNode` factory mirroring
`makeBuildNode`) and its test.

## Requirements

- [ ] `makeReviewNode(config)` returns an `EngineNode`, parameterized by prompt
      template, axis, and output key — the `makeBuildNode` shape, so lanes
      compose it the same way.
- [ ] The agent's `allowedTools` EXCLUDES every workspace-mutating tool. Derive
      the deny list from the allowlist `build` uses (`src/pipeline/nodes/build.ts`)
      rather than re-declaring one, so a tool added to build cannot silently
      become writable here.
- [ ] A test asserts the options passed to the injected `AgentQuery` carry NO
      write tool. This is the R4 test and it must assert on the OPTIONS BAG,
      not on observed behaviour — proving the capability is absent, not merely
      unused on one run.
- [ ] The node reads the run's diff via the existing `workspace.exec` seam
      (`git diff <base>...HEAD`), passed into the prompt. No new subprocess
      plumbing (Art. VIII).
- [ ] The agent's final message is parsed by `parseVerdict` (adw-m9-01). A
      parse failure is a node `fail` naming the problems — NOT a silent empty
      verdict, which would read as "no findings" and let bad work through.
- [ ] The verdict is patched onto `ctx.data` under the axis' own key, and the
      node journals `node-end` with a typed `details` bag carrying it (R7,
      Art. VI).
- [ ] Errors carry `ticketId` + node name (Art. IX).

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first, with a FAKE `AgentQuery` (Art. I — zero tokens).
- [ ] The R4 test above, asserting on the options bag.
- [ ] A parse-failure test proving the node fails rather than returning empty.

## Out of scope

The two axis prompts and their parallel wiring (adw-m9-03). Routing on the
verdict (adw-m9-04). Lane wiring (adw-m9-06).
