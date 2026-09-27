---
id: adw-cost-01-bound-what-a-tool-call-may-inject
type: feat
status: done
priority: 1
review: true
created: 2026-09-21
caps: {minutes: 180, turns: 900}
depends: [adw-cost-00-the-free-cuts, adw-bug-19-a-denied-tool-call-tells-an-unattended-agent-to-wait-for-a-human]
attempts: [{"runId":"adw-cost-01-bound-what-a-tool-call-may-inject-1789953612869","branch":"adw/adw-cost-01-bound-what-a-tool-call-may-inject","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-cost-01-bound-what-a-tool-call-may-inject-1789953612869/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-cost-01-bound-what-a-tool-call-may-inject-1789976422116","branch":"adw/adw-cost-01-bound-what-a-tool-call-may-inject-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-cost-01-bound-what-a-tool-call-may-inject-1789976422116/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/104","provider":"claude","model":"sonnet"}]
---
# P1 — the single biggest lever: stop tool output entering context whole

> **Evidence:** `ai_docs/2026-09-21-token-audit-FULL.md` §3.
> **Modelled saving: 1.02–1.53B tokens — 33–50% of the bill.**
> Larger than every other lever in the audit combined.

## The diagnostic this ticket exists for

The audit set out to shorten prompts. Measuring killed that idea and found
the real one:

| node | prompt on disk | context/turn | prompt as % of context |
|---|---|---|---|
| **`test`** | 9,001 B ≈ **2.2K tok** | **470,111** | **0.5%** |
| `build` | 7K–57K B ≈ 1–14K tok | 330,944 | ≤4% |
| `plan` | 6,459 B ≈ 1.6K tok | 75,275 | 2% |
| `repair` | (resumes build) | 389,784 | — |

> **96–99.5% of what these nodes re-read on every turn is accumulated tool
> output** — file reads, suite stdout, greps. Not the prompt. The target's
> `context:` files total ~3K tokens. **Prompt engineering cannot touch the
> 75% of the bill that is `build` + `test`.**

`build` alone is **1.562B cacheRead = 52% of everything**; `test` is **686M =
23%**.

## The shape of the fix already exists

`src/pipeline/suite-guard.ts` (ticket `adw-perf-04`) is a **pure classifier
plus a PreToolUse hook** that denies a bare `bun test` mid-stage, because bare
full-suite invocations were **80.3% of all tool time** across 50 runs. It is
the correct mechanism aimed at exactly one command. **This ticket widens it
from one command to the general class**, reusing that seam rather than
inventing a second one.

## AMENDED 2026-09-21 — this ticket was aimed at the smaller half

> **Evidence:** `ai_docs/2026-09-21-rtk-and-portal-assessment.md`.
> Measured across **207 factory sessions**, by what actually reaches the
> model's context (`tool_result` blocks in the transcript — *not* the
> PreToolUse hook payload, which is ~10x richer and misleading):
>
> | tool | share of injected tool output |
> |---|---|
> | **`Read`** | **72.1%** |
> | `Bash` | **23.1%** |
> | `Grep` · `Edit` · `Write` · `Glob` | 4.7% |
>
> The original R1 generalized `suite-guard` — a **Bash** hook. The premise
> ("`test`'s prompt is 0.5% of its context") still holds; the conclusion did
> not. **The mass is `Read`, three times over.** Requirements are re-ordered
> below; the original R1 survives as R3.
>
> **Shares above are of TOOL OUTPUT, never of the 3.049B bill.** The bridge
> between them reconciles only to 1.64x on a single session — good enough to
> confirm the mechanism, not to quote a percentage of the bill.

## AMENDMENT 2 (2026-09-21) — R1 as written is NOT BUILDABLE on this SDK

> The blocked run `…-1789953612869`'s own `plan` stage verified against the
> installed `@anthropic-ai/claude-agent-sdk@0.3.209` types that
> **retroactively shrinking an already-committed `tool_result` mid-session has
> no callable mechanism.** Only the **current** tool call's output can be
> replaced, via `PostToolUseHookSpecificOutput.updatedToolOutput`.
>
> **The saving survives; the direction reverses.** Instead of evicting the
> *earlier* read, suppress the *duplicate* read at the moment it happens:
> when the agent reads a path it has already read this session, replace
> **that** call's output with a stub pointing at the earlier one. Same 49.6%,
> same 1,758,118 tokens, and it uses the mechanism the SDK actually exposes —
> the one R2/R3 need anyway.
>
> R1 below is rewritten accordingly. **Do not attempt retroactive eviction.**
>
> This amendment was previously carried only in this build's own run
> context/PR narrative, never landed on the ticket itself — a silent
> divergence per the Amendment rule. Landed here as the actual fix.

## Requirements

- [ ] **R1 — suppress a duplicate read at the point of the call. Do this
      first; it is nearly free.**
      **49.6% of all `Read` calls re-read a path already read in the same
      session** — 3,648 reads, 1,838 distinct paths, **1,758,118 tokens of
      pure duplication = 24% of all injected tool output.** When the agent
      re-reads a path already read this session, replace **that call's**
      output (via `updatedToolOutput`) with a one-line stub naming the path
      and pointing at the earlier read. No third-party binary, no delegation,
      no network, no latency. It is the audit's own Cluster A, built on the
      seam the SDK actually exposes.
- [ ] **R1a** — never suppress a read that follows an `Edit`/`Write` to that
      path: the file has changed and the agent needs the new content. Track
      writes per path and reset the seen-set on any mutation. If that cannot
      be decided safely, suppress only when the content is byte-identical to
      the earlier read, and say so in the PR body.
- [ ] **R2 — bound what a single `Read` injects.** Mean read is **1,439
      tokens**; the tail is what hurts. Over a threshold, inject a head plus
      the path and byte count, leaving the agent to `grep`/`sed` for more.
- [ ] **R3 — a Bash-output budget at the harness boundary.** Any tool result
      over a configured threshold (start at ~4KB) is written to a file in the
      workspace and replaced in-context by **the path plus a bounded head**
      (start at 20 lines) and the byte count. The agent can `grep`/`sed` the
      file if it needs more — it just cannot inhale it by default.
- [ ] **R3a — the Bash digest must never drop the load-bearing line.** A naive
      `head` loses the failing assertion at the bottom of a suite run. For a
      recognised failure format the digest is **structured, not truncated**:
      the failing test's identity and assertion survive by construction.
      `adw-gates-03` (pytest failures are named) is the precedent for
      per-runner parsing; reuse it, do not fork it.

      > **Consider adopting `rtk` instead of building R3.** It is a Rust
      > PreToolUse proxy covering `bun test`, pytest, vitest and jest —
      > every runner our targets use — with an `rtk recall <id>` escape
      > hatch that re-fetches full output without re-running. **Scope it to
      > agent-invoked commands only, never the gate path:** `adw-gates-03`
      > parses pytest output, `suite-guard` already hooks PreToolUse on the
      > same command (ordering is undefined, and RTK's rewrite may route
      > around our guard), and `rtk init -g` is global. **Verify `rtk recall`
      > works inside an SDK session before relying on it** — if it needs a
      > TTY, R3 must spill to disk instead.
- [ ] **R4 — pure classifier, injected edge.** The decision (digest / pass
      through) is a pure function with no I/O, no clock — the idiom
      `suite-guard.ts`, `retry-policy.ts` and `profiles.ts` already establish
      in that directory. Only the writer touches the filesystem.
- [ ] **R5 — per-target threshold, factory default.** A target may raise or
      lower it; absent, the factory default applies. A target whose tests
      legitimately print a lot must not be forced to guess.
- [ ] **R6 — the digest is journalled.** A run must be able to answer "what
      did the agent actually see here" after the fact. Record bytes-in,
      bytes-injected and the spill path.
- [ ] **R7 — no lane, node or prompt changes.** This is a transport change
      beneath the agent, not a change to what any stage is asked to do.

## AMENDMENT 3 (2026-09-21) — the re-measurement Verify bullet's method is not implementable as literally written, and the substitute is weaker in one named way

> The Verify bullet below originally asked to re-measure read-duplication
> "the same way the assessment did (`tool_result` blocks in the transcript,
> by `file_path`)". Parsing that transcript for real means correlating a
> `tool_use` (carrying `input.file_path`) with its matching `tool_result` by
> `tool_use_id` inside the raw per-session `.jsonl` transcript the Claude
> Agent SDK writes (`transcript_path`, surfaced today only via
> `ClensHooksConfig.transcriptFetch`) — a shape this repo fetches and
> archives (`.clens/transcripts/`) but has never had to parse, unlike the
> flattened `tool_name`/`tool_input`/`tool_response` PostToolUse hook payload
> this ticket's own classifiers already consume. Writing and trusting a new
> parser for that shape, under this ticket's own time budget, was judged a
> worse risk than the alternative below.
>
> **Substitute:** `scripts/measure-tool-output-digest.ts` re-measures off
> this mechanism's OWN R6 journal events instead — "of every Read/Bash call
> R1/R2/R3 classified, how many did it suppress or truncate, and by how
> many bytes". This is **not** the same measurement and is weaker in one
> load-bearing way, stated plainly rather than left implicit: it grades the
> mechanism using data the mechanism itself produced, so a classifier bug
> that under- or over-fires would not be caught by this script the way an
> independent transcript re-scan would catch it. It answers "is the
> mechanism doing what it thinks it's doing", not "does the transcript's
> real duplication rate match the assessment's 49.6% baseline". A true
> transcript re-scan remains the more rigorous check and is left as
> follow-up work, not performed here.
>
> The Verify bullet below is rewritten to name the actual script and this
> caveat, rather than leaving the substitution undocumented in a source
> comment only.

## Verify

- [ ] Red test first (Art. I): the same path read twice is injected twice
      today. RED until R1.
- [ ] Red test for R1a: a read separated from its earlier twin by an `Edit`
      to that path is **not** evicted.
- [ ] Red test: a tool result of N KB is injected whole today; RED until R2/R3.
- [ ] Red test: a suite run whose failing assertion is in the **last** line
      still carries that assertion into the digest. This is R3a's teeth and
      the test that stops this ticket trading correctness for tokens.
- [ ] Red test: a result under the threshold passes through byte-identical.
- [ ] Red test: the pure classifier is exercised without touching disk.
- [ ] **Measured, and this is the acceptance bar:** run one real ticket
      end-to-end and compare `build`/`test` context-per-turn against the
      audit's **330,944 / 470,111** baseline via
      `bun scripts/run-autopsy.ts 10`. Quote both numbers in the PR body.
      A change that does not move them has not worked.
- [ ] **Re-measure the read-duplication rate** via
      `bun scripts/measure-tool-output-digest.ts` against this mechanism's
      own R6 journal events (AMENDMENT 3 — not a transcript re-scan; see the
      caveat above). Baseline to beat: **49.6% repeat reads, 1,758,118
      tokens.**
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- **Making `test` non-agentic entirely** (a deterministic runner executes, a
  stateless call classifies). That is the extreme form of this idea and the
  audit's Cluster A/F — it deletes 23% of the bill outright but changes what
  the node *is*. Land the digest first, measure, then decide.
- Turn caps and batching — `adw-cost-02`.
- The review family's diff transport — `adw-cost-03`. Its cost mechanism is
  the opposite one (its prompt **is** the cost); do not conflate them.
- **R3's spill file being agent-reachable under `container`/`remote`
  isolation.** `makeToolOutputSink` writes host-side via `node:fs`, under
  `<runsRoot>/<runId>/tool-output/`. That filesystem is shared with the
  agent's own `Bash` tool only for `worktree` isolation — `container` has no
  writable mount of the host, and `remote` (e2b) runs on a different machine
  entirely. Landed: the digest only claims "grep/sed the spill file" when
  `spillReachableByAgent` (resolved from `--isolation`) says so; for the
  other two kinds it says plainly that the file is host-side, operator-only.
  Making the spill itself reachable in those kinds — writing it through
  `Workspace.exec` instead of the host filesystem — needs the digest's
  `PostToolUse` hook to be composed per-node with a live `Workspace`
  instance, not once per stage at lane-construction time before any
  workspace is provisioned (`makeAgentNodeConfig` runs before `provision`).
  That is a larger structural change than this ticket's scope; deferred.
- **Carrying `ReadDedupState` (R1/R1a) across a `resumesSession` boundary**
  (e.g. `repair` resuming `build`'s SDK session, `nodes/build.ts`). Today's
  implementation resets dedup state per node invocation, so a re-read
  spanning that boundary is not suppressed even though it is, at the SDK
  level, the same ongoing session. `resumesSession: {from}` is the existing
  seam that already threads `ctx.data.sessionId` this way and would be the
  right carrier; wiring it needs the dedup state to live somewhere reachable
  from node code (`build.ts`) rather than only inside the
  `wrapHooksWithToolOutputDigest` closure built once per stage at lane
  construction. A naive fix (share state across every stage of a run
  unconditionally) would be wrong: `plan` and `build` do not resume one
  another's session, so blanket sharing would suppress genuine first-time
  reads in a session that never saw them. Deferred as a follow-up, not
  silently accepted.

## A note for whoever builds this

The audit's five independent ideation frames — speedrunner, hardware
engineer, inversion, remove-the-assumption, logistics — **all five**
generated this idea without seeing each other's output. That convergence is
the reason it is P1.
