---
id: adw-sysprompt-01-own-the-system-prompt
type: feat
status: done
priority: 2
created: 2026-09-11
depends: []
attempts: [{"runId":"adw-sysprompt-01-own-the-system-prompt-1789205999328","branch":"adw/adw-sysprompt-01-own-the-system-prompt","workspace":"/Users/silouane/adw-factory/runs/adw-sysprompt-01-own-the-system-prompt-1789205999328/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# The factory's agents run with an EMPTY system prompt — make that a decision

> Minted 2026-09-11. Found while auditing what the observability FE could
> persist. This is **not** an FE ticket and must not be folded into one; it is
> a factory-quality finding that happens to have surfaced there.

## The finding, empirically settled

`src/pipeline/nodes/build.ts:355-379` (`baseOptions`) never sets
`systemPrompt`. A grep for `systemPrompt|appendSystemPrompt|customSystemPrompt`
across `src test prompts` returns **zero** matches.

The assumption was that omitting it yields Claude Code's default system prompt.
**It does not.** In the bundled SDK 0.3.209 (`sdk.mjs`, function `ik`):

```js
if (i === void 0) p = "";                    // OMITTED → empty string
else if (typeof i === "string") p = i;
else if (Array.isArray(i)) p = i;
else if (i.type === "preset") f = i.append;  // preset → p stays UNDEFINED
```

The preset is signalled by the **absence** of a `systemPrompt` in the
initialize payload; omitting the option instead sends an explicit `[""]`.
Omitted and preset are different payloads, and omitted is the empty override.

**Measured, reproduced twice, same trivial prompt (`"Reply with exactly: ok"`,
model `sonnet`), total prompt tokens (`input + cache_creation + cache_read`):**

| Config | Total prompt tokens |
|---|---|
| A — `systemPrompt` omitted (**what the factory does today**) | **35 839** |
| B — `systemPrompt: {type:"preset", preset:"claude_code"}` | **43 675** |
| C — `systemPrompt: ""` (explicit empty) | **35 839** |

A and C are **identical**, and on the second pass C read A's cache prefix
entirely (`cache_read=35837`) — proof they produce the same prompt, not a
coincidence of totals. B carries **7 836 more tokens**: Claude Code's system
prompt, which **all 55 banked runs did without**.

## Why this is a decision, not a bug report

It has never been decided either way — it is an accident of an SDK default.
Both directions are defensible and the evidence genuinely cuts both ways:

**For keeping it empty:** 9 factory PRs merged without it. The factory supplies
its own 4–30 KB task prompts, which may already cover what the preset covers.
It saves ~7.8 K tokens on every turn of every agent node — real rate-limit
headroom, which `ai_docs/model-routing.md` records as the actual scarce
resource.

**For adopting the preset:** the preset carries tool-use guidance, conventions
and conciseness rules the factory's own prompts do not restate. The §6 metric
"tickets reaching in-review without blocking" sits at **60% per run** — below
its 80% target — and no one has tested whether the missing 7.8 K is a cause.

**The load-bearing point is ownership**, regardless of which wins: the factory
must *set* the value explicitly so it is a recorded choice rather than an SDK
default that can change under a dependency bump.

## Requirements

- [ ] `baseOptions` sets `systemPrompt` **explicitly** — never by omission.
- [ ] The value is configurable per target (`targets/*.json`) with an explicit
      default, so the choice is data, not a literal buried in a node.
- [ ] The resolved choice is journaled on the agent node's `node-end`, so a
      run's system-prompt provenance is answerable from its journal alone.
- [ ] Tests pin **both** paths — preset and empty — and assert the option is
      present in the query options either way. A test asserting only the
      current default would re-freeze the accident.

## Verify

- [ ] The SDK receives an explicit value in both configurations (injected
      query seam; no live call needed).
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] **Operator-gated, BILLED:** one real run per configuration against the
      same ticket, comparing outcome and token usage. Without this the choice
      is still a guess — just a recorded one.

## Out of scope

- A custom hand-written system prompt. This ticket is preset-vs-empty and
  ownership of the switch; authoring bespoke agent instructions is separate
  work with its own evidence bar.
- Codex parity — that path has no system-prompt concept at all
  (`src/codex-query.ts:590` destructures only `cwd, model, resume, signal, env`).

## Run log

_(empty)_

## Run log

**2026-09-12 — run `adw-sysprompt-01-own-the-system-prompt-1789205999328`
(worktree, claude) BLOCKED at 44.3 min: the liveness watchdog tripped on
`build` — "no SDK stream message for 10 minutes".** `plan` completed green
(16.7 min). Last tool activity was a `Read` at 12:09:01; then silence.

**Two hypotheses were checked and BOTH were wrong** — recorded so nobody
re-chases them:

- **Not rate limiting.** The capture holds 40 "rate-limit" and 16 "429"
  matches, but every one is the agent *reading adw-factory's own source*,
  which documents rate-limit failure policy heavily. No real API error.
- **Not a blocking subagent.** All 28 `Task*` events are `TaskCreate`/
  `TaskUpdate` at 0.003 s — the todo list, not spawned agents. (The `par-01`
  run *did* have 5-minute blocking `TaskOutput` waits, which is why this was
  worth ruling out.)
- **Not concurrency.** The silence window began ~13 min AFTER the concurrent
  `adw-caps-01` run finished. The two overlapped 11:40–12:01 without incident.

**The cause is not recoverable from the artifacts, and that is the finding.**
The capture records hook events, so it shows what the agent *did*, never what
it was doing between calls. The factory cannot distinguish "the model is
generating something long" from "the transport is dead" — both are silence.
The watchdog behaved correctly (Art. V): it bounded the stall instead of
letting the run hang.

Partial work in that run's worktree: 8 files, +317/−5, `build` failed
mid-node. **Not salvaged** — unlike `adw-par-01`, this is a half-built change,
not a finished one stranded before the gates. Reset to `queued` for a clean
re-run; a one-off undiagnosable stall is most likely transient.
