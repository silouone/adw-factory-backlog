---
id: adw-cost-00-the-free-cuts
type: chore
status: done
priority: 1
review: false
created: 2026-09-21
caps: {minutes: 90, turns: 400}
depends: []
attempts: [{"runId":"adw-cost-00-the-free-cuts-1789951194975","branch":"adw/adw-cost-00-the-free-cuts","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-cost-00-the-free-cuts-1789951194975/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/102","provider":"claude","model":"sonnet"}]
---
# P0 — the three cuts that cost nothing to make

> **Evidence:** `ai_docs/2026-09-21-token-audit-FULL.md`. Every figure is
> reproducible via `bun scripts/run-autopsy.ts first-review`.
> **Layer rule:** this ticket holds only changes with **no model-quality
> tradeoff and no measurement needed first**. Anything requiring an A/B or a
> judgement call belongs in `adw-cost-02` or later.

## Why these three together

The audit found the bill is **98.30% cache reads** — 2.997B of 3.049B tokens
across 64 runs, because every assistant turn re-reads the whole accumulated
context. Three defects feed that directly and none of them buys anything.

---

## R1 — the skill/slash-command lock does not take effect

`src/pipeline/nodes/build.ts:832` passes **`skills: []`**. The SDK's own
`system/init` report, journalled on every agent node, says otherwise:

| lock | asked for | resolved |
|---|---|---|
| `tools` | `AGENT_TOOLS` | **6** (3 on review) — works |
| `skills` | `[]` | **17** — ignored |
| slash commands | never locked | **46** |

The cause is named in `build.ts`'s own comment beside `AgentQueryOptions`:
**`settingSources` is not one of the fields this seam forwards**, so it rides
the SDK default and the operator's personal `~/.claude` skills and commands
load into every factory session.

**Measured cost of the whole preamble:** a factory session's first turn is
`cacheWrite 12,675 + cacheRead 11,239` = **23,914 tokens of prefix** before
any work. That run's own assembled prompt was 10,781 bytes ≈ 2,695 tokens, so
the Claude Code preamble is **≈21,219 tokens — 8× the factory's own prompt**.
Across 12,557 turns that is **266M tokens = 8.7% of the bill**.

- [ ] **R1** — forward `settingSources` (or the equivalent current SDK
      mechanism) so `skills: []` is honoured and no slash commands load.
      `adw-tools-01`'s three locks (`tools`/`strictMcpConfig`/`skills`) become
      four. **Do not** change `systemPrompt` here — that is `adw-cost-04`.
- [ ] **R1a** — the journalled `agentConfig` must show `skills=0` and
      `slashCommands=0`, so the fix is verifiable from the run record and not
      just from source.

---

## R2 — a repair loop that has stopped converging still buys two more rounds

Five runs since the review lane existed died at `node "gates" exhausted 3
repair rounds`, costing **$136.92**, of which **$53.05** is the repair rounds
themselves.

`sabado-17-one-request-cannot-freeze-the-api`, gate vector per round:

```
backend-typecheck:pass … frontend-build:pass  test:fail
backend-typecheck:pass … frontend-build:pass  test:fail
backend-typecheck:pass … frontend-build:pass  test:fail
backend-typecheck:pass … frontend-build:pass  test:fail
```

**Four identical outcomes. Rounds 2 and 3 bought nothing.**

This is **not** the reviewer-amnesia defect `adw-review-03` fixed. `repair`
**resumes the build session** (`options.resume = sessionId`), so the agent
does remember its prior attempts — which is exactly why a repeated signature
is trustworthy: a remembering agent producing an identical result has told you
it is stuck.

- [ ] **R2** — fingerprint the gate outcome (gate names × outcomes, plus the
      failing-test identity where the gate reports it) and stop the repair
      loop when a fingerprint repeats, instead of consuming the remaining
      rounds.
- [ ] **R2a** — the terminal reason must say the loop stopped on a repeated
      signature and name it. A ceiling and a livelock must not read the same.
- [ ] **R2b** — a fingerprint that *changes* between rounds keeps today's
      behaviour exactly: the loop runs to its full 3.

---

## R3 — the `test`-artifact failure discards a completed build

Four runs, **$113.25**, all identical:

```
node "test": no artifact at .adw/artifacts/test.md — the stage must WRITE
its output there (adw-bug-07)
```

On `sabado-24`: `plan` 57 turns, `build` **352 turns over 65 minutes** with
its artifact written correctly — then `test` failed **28.7 seconds in with
zero usage recorded**, and all of it was discarded.

**`dispatch` is the pattern to copy**: four runs in the same window were
refused before a session opened and cost **$0.00**.

> **Investigate before fixing.** `prompts/feature-test.md:72` *does* instruct
> the write; `plan.md` (40 KB) and `build.md` (24 KB) *were* written in that
> same run through the same path. The sabado repo has no `.adw/` directory
> while adw-factory has `.adw/artifacts/.gitkeep` committed — but that alone
> does not explain it. A 28.7s node with **zero** usage looks like the session
> dying before producing anything, with the artifact check reporting the
> symptom. `.clens/sessions/` is **not** a safe source here — it holds foreign
> sessions per `run-scorecard.ts`'s own header.

- [ ] **R3** — establish the actual cause, and state it in the PR body.
- [ ] **R3a** — whatever the cause, the precondition is checked at
      `provision`, so the run is refused **before** `plan` spends a token.
      A target that cannot receive an artifact is a config error, not a
      65-minutes-later discovery.

---

## Verify

- [ ] Red test first (Art. I) for each of R1/R2/R3a, confirmed failing.
- [ ] **R1:** a run's journalled `agentConfig` reports `skills=0`,
      `slashCommands=0`; `tools` unchanged at 6 (3 for review).
- [ ] **R1 measured:** first-turn `cacheWrite + cacheRead` on a new run is
      materially below **23,914**. Record the new figure in the PR body.
- [ ] **R2:** a fixture whose gate fingerprint repeats stops after round 1
      with a reason naming the repeat; one whose fingerprint changes still
      runs 3.
- [ ] **R3a:** a target missing the artifact path is refused at `provision`
      with `EXIT_REFUSED` and zero agent turns.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.
- [ ] `bun scripts/run-autopsy.ts 10` after the next few runs; quote the
      before/after preamble figure.

## Out of scope

- `systemPrompt: "empty"` — `adw-cost-04`. This ticket keeps the preset.
- Any change to profiles, effort or model — `adw-cost-02`.
- Tool-output digestion — `adw-cost-01`.
