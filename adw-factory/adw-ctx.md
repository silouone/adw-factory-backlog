---
id: adw-ctx
type: epic
status: queued
priority: 1
created: 2026-09-28
depends: []
children:
  - adw-ctx-01-the-tool-output-digest-has-never-applied
attempts: []
---
# EPIC — A node must not burn its run re-reading the same files: context hops carry state, and they are bounded

**Operator rule (2026-09-28): we can't have context exhaustion ending work.**
Children beyond `-01` need a short spec: hop resume content and per-node
budgets touch `build.ts` hop mechanics and Art. V caps.

## What happened (adw-learn-04, forensics of the 9 SDK transcripts)

`plan` ran **50m26s** for a small ticket. The board showed one bar with no
tool dots until the end.

- **Where the time went:** about 96% model generation (≈103 tok/s, fit
  R²=0.98), 3% tool execution, about 1 min between hops.
- **Output:** ≈265K output tokens, of which ≈240K (~90%) were **thinking**
  (229 blocks, content redacted, so the share is a subtraction). That is
  30–44K thinking tokens per session at effort `high`.
- **What it produced:** the same analysis **eight times**. The context hit
  150K each time, and every hop opened a fresh session that knew nothing.

| cause | verified evidence |
|---|---|
| the digest never shrank a Read | See `adw-ctx-01`. 1.51M chars of raw file content reached the model; the journal claims ~92K |
| the same files re-read every hop | `engine.ts` read 17× across all 8 sessions (543K chars), `journal.ts` 9× (389K), `lanes/shared.ts` 10×, `build.ts` 20× |
| the hop resume was empty | plan edits no files, so the synthesized resume (`git status` + diff stat) is "No files have been changed yet" (774 bytes, prompts 0002–0008). The checkpoint was first written in session 8 |
| no bound on hops or node share | plan used **290 of 500 turns** (58%). Build hopped 4× at the 60-turn cap (`build.ts:1893`, shrinking per hop at `:1134-1137`) and its 5th session got 15 turns, then breached at 16 |

Plan alone cost ≈$16. Two reporting bugs hid all this:

- **Usage undercounted about 120×.** The journal records plan output as
  2,205 tokens against 265,104 real, because live usage is taken from the
  first streamed message per call (`build.ts:1856-1865`, mechanism
  inferred). The context-cap check itself is sound.
- **Board dots come from the last hop only.** `gantt.ts:245-256` sets a
  block's `sessionId` from the `node-end` usage (the final session); a
  running block has none (`:273-285`). The capture data for all 9 sessions
  is present (`capture.ts:185`).

## Direction

1. **`adw-ctx-01`: make the digest real.** Biggest lever, verified, no spec
   needed.
2. **A hop resume carries what the session learned:**
   - files read, with line ranges
   - the last assistant text
   - an explicit "don't re-read these" list
   - Nudge the agent at about 110K (`additionalContext`) to write its
     checkpoint before the cap, not after.
3. **Bound it.**
   - At most 2–3 hops per node.
   - After two empty synthesized resumes, stop with a named reason; don't
     hop again.
   - A per-node share of the run's turns (e.g. plan ≤ 20%), so one stage
     can't starve the next.
4. **Right-size plan.** Effort `medium` or a thinking budget for `plan`, and
   a prompt that requires Grep and `offset`/`limit` Reads (one full
   `engine.ts` read is about 25K tokens).
5. **Honest telemetry.** Final per-message usage (not first-chunk), and
   board dots from every capture session inside a block's window, including
   the running one.

## Done when (epic)

- Re-running `adw-learn-04` as is, `plan` finishes in one session or hops at
  most once, reading each file at most once per session.
- The journal's plan token count matches the transcript within 5%.
- The board shows plan's tool dots across the whole bar while it runs.
