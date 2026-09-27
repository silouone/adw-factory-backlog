---
id: adw-tools-01-declare-the-agent-toolset
type: feat
status: done
priority: 1
created: 2026-09-13
depends: []
attempts: []
---
# The agent's tool surface is inherited from the host, not declared by the factory

> Minted 2026-09-13 from the turn-burn investigation
> (`ai_docs/2026-09-13-turn-burn-root-cause.md`). This is the finding that most
> directly contradicts the repo's own thesis: *"a software factory is a bet that
> the second run should look like the first."*

## Evidence

Measured against the 465-turn build capture
(`runs/adw-flaky-01-container-tests-contention-1789228426083/.clens/sessions/9ff334b1-*.jsonl`),
pairing `PreToolUse` to `PostToolUse` by `tool_use_id`, and against a live
one-turn probe through `liveQuery` with the real `AGENT_ALLOWED_TOOLS`.

**1. `AGENT_ALLOWED_TOOLS` restricts nothing.**

```ts
export const AGENT_ALLOWED_TOOLS = ["Bash(bun:*)", "Bash(bunx:*)"];
// "…no git (the factory owns commits) and no arbitrary shell."
```

| | |
|---|---|
| allowlisted Bash calls (`bun`/`bunx`) | 48 |
| **NON-allowlisted Bash calls that RAN** | **284** |

Head words across 346 Bash calls: `tail` 151, `grep` 72, `bun` 44, `docker` 41,
`git` 14, `sed` 4, `rm` 4, `bunx` 4, plus `nohup`, `sh`, `bash`, `for`, `time`.
The SDK's `allowedTools` is an **auto-approve** list, not a restriction.

Two things this voids:
- The five prompts asserted *"git is not in your allowed tools — attempting it
  only wastes turns."* (Corrected 2026-09-13; the rule is now stated as a rule.)
- `printenv` was removed in adw-m6-03 as *"a gratuitous env-exfil primitive"*.
  It was removed from a list that never contained anything. `sh -c 'env'` is one
  call away, and the local-spawn `sanitizeAgentEnv` denylist is the only real
  boundary.

**2. The toolset is whatever the host machine's Claude config provides.**

The probe's own answer to "list your tools":
`Agent, Bash, Edit, Read, ReportFindings, ScheduleWakeup, Skill, ToolSearch,
Workflow, Write, advisor`. The banked capture corroborates: `TaskCreate`,
`TaskUpdate`, `Monitor`, `ToolSearch` all appear as real calls. **None of these
are chosen by the factory.** A clone on another machine gets a different agent.

**3. Two concrete costs of that.**

- **`Grep`/`Glob` are absent** → that build spent **72 Bash calls on `grep`**.
  A dedicated Grep tool is one call; `grep` via Bash is one call plus the
  Bash-permission path, and returns raw untruncated output into context.
- **`Agent` and `Workflow` are present** → a build agent can spawn subagent
  fleets, and `consumeAgentStream` deliberately does **not** count subagent
  turns (`parent_tool_use_id`) against `ctx.caps.turns`. That spend is invisible
  to the ceiling that is supposed to bound the run.

## Requirements

- [ ] Establish the actual current surface first: enumerate what the SDK
      resolves for an agent node on this machine, and record it. **Do not
      "fix" it before it is measured** — the numbers above are from one
      capture and one probe.
- [ ] Declare the tool surface in the factory rather than inheriting it. SDK
      0.3.209 exposes `Options.disallowedTools` (`sdk.d.ts:1356`) and
      `canUseTool` (`:1341`); `allowedTools` alone cannot do this.
- [ ] Decide, explicitly and with the reason recorded, whether `Grep`/`Glob`
      are added (measure the turn delta) and whether `Agent`/`Workflow` are
      removed or their turns counted. Either is defensible; silently inheriting
      both is not.
- [ ] Make the code comments true. `AGENT_ALLOWED_TOOLS`'s doc comment claims
      a restriction that does not exist.
- [ ] Do **not** reach for a blanket `docker`/`git` ban: `adw-flaky-01`'s own
      ticket legitimately needs `docker`, and whack-a-mole on command names is
      not a boundary.

## Verify

- A unit test pins the exact options the agent nodes send the SDK, including
  the restriction field, so the surface cannot drift silently again.
- A live one-turn probe on a clean checkout lists the same tools as the same
  probe on this machine.

## Out of scope

Counting subagent turns against the ceiling is a separate decision with its own
evidence (M6 run-4: a productive 48-file run was killed at "201 turns" of mostly
subagent chatter — which is *why* they are uncounted). Name it, do not change
it here.
