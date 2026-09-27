---
id: adw-sysprompt-02-preset-default
type: chore
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: []
---
# The default system prompt must be the Claude Code preset, not an empty string

`adw-sysprompt-01` made the system-prompt choice explicit and deliberately
froze the default at `"empty"` to keep all 55 banked runs byte-identical. It
named the bar for ever changing it:

> *"this ticket is about ownership of the choice, not about winning the
> empty-vs-preset argument — an operator-gated live A/B (the ticket's own
> Verify) is the evidence bar for ever flipping this default."*

**That decision was taken by the operator on 2026-09-14.** This ticket records
it and flips the constant.

## Why

1. **`""` runs the model off-distribution.** The SDK ships Claude Code's own
   system prompt and the models are RL-trained against it. Per Horthy's *"Why
   Software Factories Fail"* (banked at
   `~/brain-hub/transcripts/youtube-Ib5GBkD555M.txt`): Claude Code beat Aider
   and Codebuff, which had the same tools, because *"this was the first time
   that a model lab trained a model against the harness that they were going
   to distribute it to users in."* Sending `""` keeps the weights and discards
   the scaffolding they were trained with.
2. **The discarded scaffolding is exactly our measured failure mode.** The
   preset encodes tool-use discipline. `ai_docs/2026-09-13-turn-burn-root-cause.md`
   measures a 465-turn build spending 151 calls on `tail`, 72 on `grep`, and
   `ai_docs/2026-09-13-state-of-the-factory.md` §4 puts the whole green/blocked
   split in `build` turn count (42-70 green vs 197-465 blocked).
3. **Practice had already outvoted the default.** Both live targets
   (`adw-factory`, `clens`) set `systemPrompt: "preset"` by hand. The default
   only ever governed `claude-home`, `sabado` and `scratch` — and silently gave
   them the *more* exotic option.
4. **A default should be the boring choice.** A target whose author did not
   think about it got no system prompt at all.

## Accepted cost

The preset is ~7.8K prompt tokens per agent node, and it is **SDK-owned**: it
can change under a dependency bump, which is a real reproducibility cost for a
factory whose thesis is that the second run looks like the first. The operator
accepted this trade: `""` is paid on every turn, a bump is paid rarely and
visibly.

## Requirements

- `DEFAULT_SYSTEM_PROMPT = "preset"`.
- `resolveSystemPromptOption`'s MAPPING is unchanged — only which choice is the
  default moved. A target can still opt out with `"systemPrompt": "empty"`.
- Every test that pinned "absent → empty-string override" now pins
  "absent → the Claude Code preset object", including the end-to-end CLI
  assertion and the journal's recorded `usage.systemPrompt`.

## Verify

- `bun test test/pipeline/nodes/sysprompt-preset-default.test.ts` — red before,
  green after.
- `bun test test/pipeline test/cli.systemprompt.test.ts` — 541 pass, 0 fail.
- `bun run lint && bunx tsc --noEmit` green.
- `bun scripts/dump-prompt-system.ts` shows every target resolving to the
  preset object.

## Follow-up, NOT done here

No live A/B was run. The argument above is mechanistic and from measured turn
sinks, not from a controlled experiment on this factory. If green rate does not
improve, re-open — the knob is one constant and every target can still opt out
individually.
