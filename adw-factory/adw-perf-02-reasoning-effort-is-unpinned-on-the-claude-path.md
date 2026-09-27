---
id: adw-perf-02-reasoning-effort-is-unpinned-on-the-claude-path
type: bug
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-perf-02-reasoning-effort-is-unpinned-on-the-claude-path-1789465934339","branch":"adw/adw-perf-02-reasoning-effort-is-unpinned-on-the-claude-path","workspace":"/Users/silouane/adw-factory/runs/adw-perf-02-reasoning-effort-is-unpinned-on-the-claude-path-1789465934339/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/51","provider":"claude","model":"sonnet"}]
---
# Every Claude run inherits `effort: high` from the operator's machine — the codex path fixed this in July and the claude path never did

> Operator, 2026-09-15, looking at the tool-tick gaps on a run card:
> *"why are those gaps in the plan and build… why is the model going idle so
> often? this is absorbing A LOT of time."*
>
> **The model is not idle.** It is thinking, at an effort level nobody chose.

## 1. The gaps are real generation — rule out dead time first

Measured across **7,957 inter-tool gaps** in every banked capture:

| | |
|---|---|
| total gap ("model") time | **26.8 hours** |
| total output tokens ever produced | **5,433,690** |
| implied throughput during gaps | **56.3 tok/s** |

56 tok/s is exactly the healthy figure `adw-perf-01` measured for model-only
throughput. **The arithmetic closes: there is no dead time in the gaps.** The
model is generating, at full speed, for all 26.8 hours.

But look at the shape:

| | count | % of gaps | median | total | % of model time |
|---|---|---|---|---|---|
| gaps **> 60s** | 278 | **3.5%** | **145s** | 13.3h | **50%** |

**3.5% of the gaps are half of all model time.** And what follows them is not
a big write — it is `Bash` (176, median 145s) and `Read` (61, median 163s).
Compare the gap before an `Edit`: median **6.0s**.

So the model spends ~2.5 minutes generating, and then reads a file. That is
not composition. That is **extended thinking**.

## 2. The cause, and the codebase already knows it is wrong

Every factory session's captured config carries:

```json
"effort": {"level": "high"}
```

The factory never sets it on the Claude path. `grep -rn effort src/` returns
hits in **`codex-query.ts` only** — where it is pinned, deliberately, with the
reason written out (2026-07-19):

> **a factory run MUST be deterministic w.r.t. the user's
> `~/.codex/config.toml`.** The first live bar blocked because the global
> config carried `model_reasoning_effort = "max"` … and the argv passed no
> override, so **the run inherited it** and 400'd. Passing
> `-c model_reasoning_effort=` makes the run **independent of global config**
> (the same determinism guarantee as pinning the model).

`live-query.ts` has no equivalent. It pins `model`, `systemPrompt`, `tools`,
`skills`, `strictMcpConfig` — every other knob that could drift — and leaves
reasoning effort to whatever the operator's `~/.claude` happens to say.

**This is the same bug the codex path was fixed for, on the other provider,
unfixed.** Two consequences:

- **Speed.** `high` effort on a `build` stage that is implementing an
  already-written plan buys deliberation the plan already did.
- **Reproducibility, which is worse.** Change your global Claude Code effort
  setting and every factory run changes behaviour, with nothing in the journal
  recording why. That defeats the whole "the second run should look like the
  first" premise in the README's first paragraph.

## 3. Requirements

- [ ] **Pin reasoning effort on the Claude path**, mirroring
      `CODEX_REASONING_EFFORT` exactly (Art. VIII — same decision, same shape,
      one precedent). It rides `AgentQueryOptions` as a REQUIRED field, never
      a conditional spread, exactly as `systemPrompt` and `tools` already do
      after `adw-sysprompt-01` / `adw-tools-01`.
- [ ] **Journal the resolved effort** on the agent node's usage record,
      alongside `model`. A run whose behaviour depends on it must say so
      (Art. VI). It is already in the captured `agentConfig` — promote it.
- [ ] **Per-stage effort is the actual win, and it needs a decision.**
      `plan` is a judgement stage and probably wants `high`. `build`
      implementing an existing plan, and `repair` fixing a named gate failure,
      likely do not. The node factory already takes per-instance overrides
      (`extras.tools`, `adw-m9-02`) — the seam exists.
- [ ] **Measure, do not guess.** Run the same ticket at `high` and at a lower
      effort on `build` only; record wall clock, output tokens and whether the
      gates still go green on the first pass. Put both numbers in this ticket.
      A speed change that costs a repair round is not a speed change.

## 4. What this does NOT claim

Lowering effort is **not** obviously right. `high` may be exactly what a
factory wants for a stage that must get architecture right unattended, and a
cheaper `build` that needs an extra repair round is a net loss. The defect
being fixed here is that **nobody chose** — the value is ambient, invisible in
the journal, and differs between the two providers for no stated reason.

Pin it first. Tune it second, with numbers.

## Verify

- [ ] Two runs of the same ticket with the operator's global effort set
      differently produce the same journaled effort.
- [ ] The resolved effort appears in `just usage <runId>`.
- [ ] Before/after wall clock and token figures recorded in §3.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Related

`adw-perf-01` (three cold sessions, 31% suite) and `adw-bug-06` (the suite is
78% of tool time) are the other two throughput items. This one is the cheapest
to land and the only one that is also a **determinism** bug.
