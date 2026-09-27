---
id: adw-m1-10-lane-cli
type: feat
status: done
priority: 1
created: 2026-07-14
epic: adw-m1
depends:
  - adw-m1-02-selection
  - adw-m1-03-status-writer
  - adw-m1-05-engine
  - adw-m1-08-gates-node
  - adw-m1-09-agent-nodes
attempts: []
---
# Chore lane + `adw run` CLI

## Context

Wires every M1 piece into the runnable loop. Dispatch is a code `switch` on
`type:` — routing is degenerate with one lane (decision 6). The operator is
the scheduler (S6.1); caps default deliberately huge so repair rounds govern
(decision 12).

## Deliverables

- `test/pipeline/lanes/chore.test.ts`, `test/cli.test.ts` (first, red)
- `src/pipeline/lanes/chore.ts`, `src/cli.ts` (+ `bin` entry in package.json)

## Requirements

- [x] `lanes/chore.ts` declares the M1 node list — dispatch → provision →
      assemble-prompt → build → gates (retry → repair) — plus config:
      `rounds: 3`, `caps: {minutes: 480, turns: 200}`, `model: 'sonnet'` per
      agent node; every value per-ticket overridable via `ctx.data.ticketModel`
      + `RunOptions.ticketCaps` (S6.3, decisions 10/12). NOTE: `validate` is
      satisfied PRE-ENGINE by `parseTicket` in the CLI orchestrator, so it is
      not a lane node; the lane starts at dispatch (first journaled node,
      Art. VI). Reviewer judged this placement more correct than the sketch.
- [x] Dispatch node: code switch on `type` → lane; no agent (Art. III);
      commits `in-progress` via the m1-03 protocol before provisioning
      (S1.4, E7)
- [x] `adw run [--ticket <id>] [--isolation worktree|container|remote]`:
      exactly one ticket per invocation; node-by-node progress streamed to
      the terminal (S6.1); `container`/`remote` parse but exit with
      "not yet available" (S4.1, S4.5). Live SDK query injected in adw-m1-11;
      the full pipeline is driven head-to-tail by `test/cli.test.ts` with a
      fake query (zero tokens, Art. I).
- [x] Malformed ticket → the m1-01 error report printed, exit non-zero,
      nothing provisioned, zero tokens (S1.3)
- [x] Run outcome (green/blocked) reflected in exit code; journal written in
      all cases (S5.1)
- [x] Terminal finalizer: every `blocked`/aborted outcome ends with a
      committed `status: blocked`, and the ticket's `attempts:` entry records
      run id, attempt branch, and workspace path — no terminal path leaves the
      ticket `in-progress` (S2.3, S2.6, plan §5 as amended). The finalizer
      writes attempts as a single-line inline JSON array; `parseTicket` was
      extended to read it (round-trip closed), so a re-queued blocked ticket
      stays selectable (S3.3).

## Build protocol (Art. I)

1. Lane tests over the engine with faked I/O edges; CLI tests via argument
   parsing + orchestration seams (no live agent).
2. Red → implement → green.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

push/open-pr/sync nodes (M2), `adw status`/`adw clean` (adw-m2-05).
