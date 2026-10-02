---
id: adw-bug-40-a-tool-approval-outage-lets-agents-code-blind-165ff7
type: bug
status: blocked
priority: 1
created: 2026-10-02
caps: {minutes: 120, turns: 300}
depends: []
attempts: [{"runId":"adw-bug-40-a-tool-approval-outage-lets-agents-code-blind-165ff7-1790928351440","branch":"adw/adw-bug-40-a-tool-approval-outage-lets-agents-code-blind-165ff7","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-40-a-tool-approval-outage-lets-agents-code-blind-165ff7-1790928351440/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# When tool approval goes down, agents keep coding blind and the run blocks at gates instead of waiting

## Why

Five runs have been burned this way: adw-learn-03 and adw-merge-01 (2026-09-29), hq-05 (2026-10-01), and hq-07 twice (2026-10-02, runs `…-1790900208265` and `…-1790925359587`).

Agent sessions run with `permissionMode: "auto"` (`src/pipeline/nodes/build.ts:1552`). Any Bash call outside the allowlist goes to a model classifier for approval. While that classifier was down, every such call came back as an `is_error: true` tool_result with this text:

> `claude-sonnet-5-5 is temporarily unavailable, so auto mode cannot determine the safety of Bash right now. Wait briefly and then try this action again. If it keeps failing, continue with other tasks that don't require this action and come back to it later. …`

- **42 such denials** in hq-07's second run, spread over the build node and all 3 repair rounds.
- The text tells the model to *continue with other tasks*, and it did: it wrote ~2800 lines with Edit/Write and never formatted or type-checked them.
- `isFatalDenialText` (`build.ts:2319`) matches only the STOP denial (`/^The user doesn't want to take this action right now\b/`). The outage text counts as an ordinary retryable tool error, and no no-progress detector fires, because Edit/Write calls keep succeeding.
- Every node ends `next`. The run blocks with `node "gates" exhausted 3 repair rounds`, which names the symptom, not the cause.
- Repair spawns into the same outage, so all three rounds are paid for and none can succeed.

Each hand salvage (silou-hq PR #5, PR #9, adw-factory PR #162) found the work nearly complete and only needing a formatter and tsc.

## What to build

1. **A pure classifier**, `isApprovalOutageText(content: unknown): boolean`, beside `isFatalDenialText`.
   - It matches the confirmed text with a regex anchored at the START: `/^\S+ is temporarily unavailable, so auto mode cannot determine the safety of \w+/`.
   - The start anchor matters for the same reason as adw-bug-24: this repo tests itself, and test output can contain the phrase.
   - Leave the model name and tool name as wildcards. The outage has named `claude-sonnet-5-5` and `Bash`, and the next one may name others.
2. **A per-session counter in the SDK message loop** (the `message.type === "user"` branch in `build.ts`, the same place `isFatalDenialText` is checked).
   - It counts outage denials across the session.
   - At the threshold (`APPROVAL_OUTAGE_THRESHOLD = 3`, exported), the node ends with `requestCleanup()` and a **transient** outcome, not a `fail`. Reason: `tool approval unavailable for ticket "<id>" (node <name>): <n> denials — "<first 120 chars>"`.
   - Use the existing transient path: `kind: "retry"` under `retryTransient` (adw-resilience-01) for an instance that opted in. Opt in for build, test and repair.
   - The counter covers every agent node that runs through this loop. Check that repair (`src/pipeline/nodes/repair.ts`) does; it imports from build.ts.
3. **When transient retries are exhausted**, the run ends `blocked` with the outage reason above, not with `gates exhausted`. Journal a distinct event (`approval-outage`) carrying the node, the count and the first denial text, so `adw web` and the journal show the real cause.
4. **The workspace is never reset by this path.** The agent's partial work stays in place for the retry and for any hand salvage.

## Spec amendment (land it in the same PR)

Add the outage denial to the headless-denial section next to adw-bug-24's amendment. State three things:
- the three outcomes: STOP is fatal; "requires approval" is retryable by the model; an approval outage is a transient for the engine, never the model's to work around;
- the threshold and why: one blip is tolerated, three in a session means the model will start working blind;
- that the reason must name the outage.

## Red first (Art. I)

- `isApprovalOutageText`:
  - true for the verbatim hq-07 text;
  - true with another model and tool name substituted;
  - false when the phrase sits mid-string (inside test output);
  - false for the STOP text, the "requires approval" text, `Exit code 1`, and a non-string.
- Message loop, with a fake SDK stream:
  - 2 outage denials followed by a normal `result` → the node proceeds as today;
  - 3 outage denials → the node returns the transient outcome with the reason format above, and `requestCleanup` is called once;
  - an outage denial does NOT trip `isFatalDenialText`, and a STOP denial still fails immediately.
- Engine level: transient retries exhausted → the run ends `blocked` with the outage reason, the `approval-outage` event is journaled, and no repair round is started for it.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`
- Replay check: feed one repair transcript from `runs/hq-07-search-inspector-and-full-screen-in-preact-a2c261-1790925359587/transcripts/` through the classifier. It must trip at the 3rd denial.
