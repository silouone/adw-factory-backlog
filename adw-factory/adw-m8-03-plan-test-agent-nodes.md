---
id: adw-m8-03-plan-test-agent-nodes
type: feat
status: done
priority: 3
created: 2026-07-23
epic: adw-m8
depends: [adw-m1-07-target-config-prompt, adw-m1-09-agent-nodes]
attempts: []
---
# Reuse the `build` node for `plan`/`test` stages + `{{plan}}` threading

> Source of truth: `specs/adw-v1.1-lanes.md` (§Agent stages — still two node
> types; Impl. decisions 3, 6). Read constitution + specs first.
> **No new agent node type** — Gate III stays intact; `plan`/`test` are
> instances of the existing `build` node.

## Context (grounded in source)

- `src/pipeline/nodes/build.ts:171` — `makeBuildNode(query, config)` returns an
  `EngineNode` with a **hardcoded** `name: "build"`, reads `ctx.data.prompt`, and
  patches `{ sessionId, agentSummary }`. Mechanically a `plan`/`test` stage is
  the same: a fresh query capturing a summary — it differs only by prompt, node
  **name** (for distinct journal/spans, Art. VI), and **output key**.
- `src/pipeline/nodes/assemble-prompt.ts` — `assemblePrompt(ticket, target,
  template)`; `KNOWN_PLACEHOLDERS = [ticketBody, conventions, contextPointers]`;
  scans the template, throws on unknown placeholders (N5).
- Gates already run tests deterministically (`gates.ts`) — so the `test` stage
  **authors/expands** tests, never merely runs them (Art. III). The judgment is
  in the prompt (m8-04), not a new node type.

## Requirements

- [ ] **Parameterize `makeBuildNode`** by node `name` and output `ctx.data` key
      (default `agentSummary`), so `plan` (→ `ctx.data.plan`), `build`
      (→ `agentSummary`), and `test` are **instances of one node type** — not new
      types (Art. VIII "one representation"; Gate III intact). Existing `build`
      callers stay byte-identical (defaults preserve today's name + key).
- [ ] **`{{plan}}` placeholder**: `KNOWN_PLACEHOLDERS` gains `plan`;
      `assemblePrompt` renders `ctx.data.plan` into it (absent → empty, like the
      other slots). Signature unchanged; the unknown-placeholder scan still
      rejects typos.
- [ ] chore assembled-prompt **snapshot byte-identical** (chore templates carry
      no `{{plan}}`; the build node's default name/key are unchanged).

## Build protocol (red-first, Art. I)

1. Tests (build/repair node tests are prior art): a `plan`-named instance writes
   its summary to `ctx.data.plan`; a `build` template renders `{{plan}}`; the
   default `build` instance is byte-identical (name `build`, key `agentSummary`);
   the chore snapshot is untouched. Confirm red, present for review.
2. Implement to green.
3. Verify.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; parameterization +
plan-threading covered; existing build behavior + chore snapshot identical.

## Out of scope

Template *content* (m8-04); lane composition (m8-06/07). No new agent node type.
