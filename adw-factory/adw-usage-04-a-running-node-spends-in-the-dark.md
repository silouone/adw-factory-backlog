---
id: adw-usage-04-a-running-node-spends-in-the-dark
type: feat
status: done
priority: 1
created: 2026-09-26
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"adw-usage-04-a-running-node-spends-in-the-dark-1790473626224","branch":"adw/adw-usage-04-a-running-node-spends-in-the-dark","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-04-a-running-node-spends-in-the-dark-1790473626224/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-usage-04-a-running-node-spends-in-the-dark-1790479260068","branch":"adw/adw-usage-04-a-running-node-spends-in-the-dark-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-04-a-running-node-spends-in-the-dark-1790479260068/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/121","provider":"claude","model":"sonnet"}]
---
# A running agent node spends in the dark: show its usage live

> **Evidence, 2026-09-26, run `adw-store-01-tickets-dir-1790448708493`.** At
> 116 min elapsed the run header read `TURNS 66 · BILLED 55,697 · CONTEXT
> 3,749,428 · COST ~$2.48`. Those four figures are **exactly** the finished
> `plan` node (38 API calls, 55,697 in+out, 3,537,200 cache-read + 156,531
> cache-write). The `build` node had then been running for 85 min, with **420 API
> calls, 185,335 in+out, 176.2 M cache-read and 0.58 M cache-write**, and none of
> it was shown. The real run was about 50× the displayed cost. The operator read
> "long and cheap" and could not tell whether to interrupt. The build then died
> on an API 400 at message 947, so the node never ended and its usage was
> **never** shown.

## What happens today

An agent node's usage reaches the journal only when the node ends
(`node-end`'s usage summary, from the SDK `result` message / Codex
`turn.completed`). The board card and the run header sum finished nodes only.
A node that is running, or that dies before its `result`, contributes nothing.
The long, expensive node is exactly the one you are watching.

## What to build

Usage becomes a **live** quantity for every agent node, on both providers:

- **Emit as it streams.** While an agent node streams, the factory journals
  cumulative usage for that node. It covers API calls (distinct assistant
  message ids), input, output, cache-read and cache-write tokens, and the
  current context size (the last call's input + cache). It is
  **deduplicated by message id**, since one API call can surface as several
  assistant stream messages. The journal cadence is bounded: piggy-back on
  the existing heartbeat, or at most one usage event every N seconds, never
  one per stream message.
- **Both providers.** The Claude path reads `message.usage` per assistant
  message. The Codex path reads its turn usage events. A provider that
  reports nothing journals nothing; never fabricate zeros (Art. VI).
- **Reconcile at node end.** When `node-end` arrives, its usage summary is
  the truth for that node and replaces the live figure. A node that dies
  without a `node-end` keeps its last live figure, marked as such, so a
  blocked run still shows what it spent.
- **Render live.** The run header, the board card and the run-screen node
  drawer sum finished nodes **plus** the live figure of any running node. The
  live part is visibly marked (e.g. "live" or `~`) until reconciled.
  Also show **context now** (the current prompt size of the running node),
  which is the number that explains a slow, expensive node.
- **Cost** uses the same pricing path as today, applied to the live figures.

## Acceptance criteria

- [x] A fixture stream of a running Claude node (N assistant messages, some sharing a message id) journals cumulative usage equal to the per-message-id deduplicated sum, at the bounded cadence.
- [x] The same for a Codex fixture stream.
- [x] Projection test: a journal with a finished `plan` node and a running `build` node with live usage renders header totals equal to plan + live build, with the live part marked.
- [x] Projection test: a node that has live usage and then a `node-end` shows exactly the `node-end` figures (no double count).
- [x] Projection test: a node with live usage and no `node-end` (run ended `blocked` mid-node) still contributes its last live figure, marked as unreconciled.
- [x] "Context now" renders for a running node and disappears (or freezes, labelled) when it ends.
- [x] The board card and the run header render the same totals for the same run (`adw-fe-15`: two screens, one language).
- [x] Journal size: a 90-minute node adds at most a bounded number of usage events (state the bound in the test).

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`. Then, manually, `adw web`
during a live run: the header's billed/context/cost move while the build node
runs.

## Blocked by

None. This ticket can start immediately.
