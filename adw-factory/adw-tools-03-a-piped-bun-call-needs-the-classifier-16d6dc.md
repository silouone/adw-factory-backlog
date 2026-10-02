---
id: adw-tools-03-a-piped-bun-call-needs-the-classifier-16d6dc
type: bug
status: in-review
priority: 1
created: 2026-10-02
caps: {minutes: 90, turns: 250}
depends: []
attempts: [{"runId":"adw-tools-03-a-piped-bun-call-needs-the-classifier-16d6dc-1790937456097","branch":"adw/adw-tools-03-a-piped-bun-call-needs-the-classifier-16d6dc","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-tools-03-a-piped-bun-call-needs-the-classifier-16d6dc-1790937456097/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/183","provider":"claude","model":"claude-sonnet-5-5","rebased":"4ed59baf4902925299fb07886d985f00e4bf2302"}]
---
# A piped `bun`/`bunx` call falls outside the allowlist and needs the classifier

## Why

`AGENT_ALLOWED_TOOLS` (`src/pipeline/nodes/build.ts:301`) is `["Bash(bun:*)", "Bash(bunx:*)"]`. The intent: the commands an agent needs to check its own work never depend on the auto-mode classifier.

They do anyway, because agents pipe nearly everything. On hq-07 (`runs/hq-07-search-inspector-and-full-screen-in-preact-a2c261-1790925359587/transcripts/`), these calls were all DENIED by the classifier outage (see adw-bug-40):
- `bunx tsc --noEmit 2>&1 | head -40` (6×)
- `bun test test/web 2>&1 | grep -vE "^\s*at |^$" | tail -60`
- `bunx biome check --write src test 2>&1 | tail -80` (5×)

A prefix rule does not cover a compound command. Claude Code checks each segment of a pipeline, so the `head` / `tail` / `grep` segment goes to the classifier. While the classifier was up, the same commands went through, which hid the dependency.

This ticket shrinks how much an outage can hurt. adw-bug-40 handles the outage itself.

## What to build

1. **Measure first (R0, record the result in the PR).** Run a live SDK session in a throwaway workspace with `permissionMode: "dontAsk"`, which denies anything not pre-approved, so no classifier is involved. Use the current allowlist, then the extended one, and try:
   - `bun --version`
   - `bun --version 2>&1 | tail -1`
   - `bunx tsc --version | head -1`
   - `bun --version | grep -c .`

   Record which ones pass. If the extended allowlist does not make the piped forms pass, STOP and amend this ticket. Do not guess a rule syntax.
2. **Extend `AGENT_ALLOWED_TOOLS`** with read-only output filters only: `Bash(head:*)`, `Bash(tail:*)`, `Bash(grep:*)`, `Bash(wc:*)`, `Bash(sort:*)`.
   - **Not** `sed`, `awk`, `xargs`, `tee` or `cat`: each of them can write files or run other commands.
   - `mergeAllowedTools` stays order-preserving, and a target's own `allowedTools` (adw-tools-02) still merges on top.
3. **Tell agents in the prompt.** Add one line to the assembled agent prompt near the gate commands: run check commands unpiped, or piped only through `head`/`tail`/`grep`. Any other pipe needs an approval that may be unavailable.

## Spec amendment

Update the adw-tools-01/02 section of the spec with two points:
- the allowlist includes read-only output filters, and why: compound-command segment checking;
- the list of filters that were rejected, and why.

## Red first (Art. I)

- `AGENT_ALLOWED_TOOLS` contains the five filters and none of `sed`, `awk`, `xargs`, `tee`, `cat`.
- `mergeAllowedTools(undefined)` equals the new list; a target list is merged after it, deduplicated.
- The assembled build and repair prompts contain the pipe guidance line.

## Verify

- `bun run lint && bunx tsc --noEmit && bun run test`
- R0 results pasted into the PR body.
