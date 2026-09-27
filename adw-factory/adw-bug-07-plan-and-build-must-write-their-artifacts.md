---
id: adw-bug-07-plan-and-build-must-write-their-artifacts
type: bug
status: done
priority: 1
created: 2026-09-15
depends: []
attempts: [{"runId":"adw-bug-07-plan-and-build-must-write-their-artifacts-1789459934670","branch":"adw/adw-bug-07-plan-and-build-must-write-their-artifacts","workspace":"/Users/silouane/adw-factory/runs/adw-bug-07-plan-and-build-must-write-their-artifacts-1789459934670/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-bug-07-plan-and-build-must-write-their-artifacts-1789465894523","branch":"adw/adw-bug-07-plan-and-build-must-write-their-artifacts-2","workspace":"/Users/silouane/adw-factory/runs/adw-bug-07-plan-and-build-must-write-their-artifacts-1789465894523/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-bug-07-plan-and-build-must-write-their-artifacts-1789465894523","branch":"adw/adw-bug-07-plan-and-build-must-write-their-artifacts","workspace":"/Users/silouane/adw-factory/runs/adw-bug-07-plan-and-build-must-write-their-artifacts-1789465894523/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/53","provider":"claude","model":"sonnet","salvagedBy":"operator"}]
---
# `adw-bug-05` shipped the diagnostics and skipped the fix — the 300s cliff is still there, and it is still blocking three tickets

> **Read this first.** `adw-bug-05` merged as PR #47 and reads `in-review`.
> It changed `journal.ts`, `engine.ts` and one test — **§5 only** (journal the
> stall reason, which is genuinely useful and is done). **§4 — the actual fix
> — was never built.** `prompts/feature-plan.md` is untouched.
>
> So the ticket looks closed, the tests are green, and the cliff is entirely
> intact. This ticket is §4, scoped to nothing else.

## 1. Proof it is still there

`adw-m9-04-review-fix-loop-1789458034035`, run **after** #47 merged:

| attempt | tool calls | trailing silence before death |
|---|---|---|
| 1 | 48 | **303.1s** |
| 2 | 56 | **303.5s** |
| 3 | 60 | killed by the operator |

Against the eight stalls measured before it — 302.3s, 302.6s, 303.0s, 304.5s,
307.4s, 307.6s, 338.6s. **Ten data points, all ~303s.** The timeout is exact,
reproducible, and unaffected by anything #47 shipped.

## 2. Why the cause is a prompt, not a transport

`prompts/feature-plan.md:34` instructs the plan stage to stop calling tools and
emit one long document:

> Your final message **is** the plan artifact: it is captured verbatim and
> handed to the build agent as its plan. Make it self-contained and specific…
> the plan is your only output.

During that generation the stream carries no tool activity. Exceed ~300s and
it dies. `build` hits the same wall composing its final report — that is what
killed `adw-fe-14` at **303.0s** with a complete, green, 117-test feature in
the worktree (salvaged by hand as PR #46).

Successful agent nodes have a median trailing silence of **19s**. Stalled ones
cluster at **303s**. That gap is the whole bug.

**A failed workaround, recorded so nobody retries it:** inlining the recovered
plan into `adw-m9-04`'s body made things *worse* — the prompt tripled (5,273 →
15,977 bytes) and tool calls rose from ~13 to ~55, because the agent went and
verified the plan against the codebase. The template's "your final message IS
the plan" outranks anything the ticket body asks for. **Do not fix this from
the ticket side.**

## 3. Requirements

- [ ] **`prompts/feature-plan.md`: the plan is WRITTEN to a file**, not emitted
      as the final message. The `plan` node reads that file into
      `ctx.data.plan`. Every `Write` is a tool call, so the stream never idles
      to the cliff.
- [ ] **Written in sections, not one call.** A single `Write` of a 10K-token
      document is still one long generation. The prompt must ask for the plan
      to be built up incrementally (outline first, then each section appended),
      so no single generation approaches 300s.
- [ ] **`prompts/feature-build.md` and the other build-shaped prompts: the
      build report is written to a file too**, for the same reason. fe-14's
      303s report loss cost a finished feature.
- [ ] **The file is inside the workspace** and therefore already captured by
      the run — which makes the artifact **durable**. A stall after the plan
      exists stops losing it. Today recovery is manual, out of
      `runs/<id>/prompts/0003-build.txt` (see
      `ai_docs/2026-09-14-m9-04-recovered-plan.md`).
- [ ] **`plan` must still not touch the repo's own files.** It writes its
      artifact to a dedicated path and nothing else — the existing "make no
      changes to the working tree" rule stands for everything but that file.
- [ ] Check whether the SDK exposes a stream/idle timeout. If so raise it **as
      well** — but this fix does not depend on it, and must not.

## 4. Red tests

- [ ] A fake `AgentQuery` whose session writes a plan file and returns a SHORT
      final message → the `plan` node still populates `ctx.data.plan`, read
      from the file. This is the contract change; it fails today.
- [ ] The plan file is absent → the node fails with a descriptive error naming
      the ticket and the expected path (Art. IX), not a silent empty plan.
      An empty plan reaching `build` is worse than a refusal.
- [ ] `build`'s report path, same two cases.
- [ ] Red before any fix.

## Verify

- [ ] `adw-m9-04-review-fix-loop` completes. It has failed **four times** on
      this, and it gates `adw-m9-06` and therefore the whole review lane.
- [ ] `adw-bug-02-run-screen-axis-and-liveness` completes. Also blocked twice
      on this.
- [ ] Median trailing silence on agent nodes stays near 19s, measured from
      captures across at least 3 runs.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Why P1

It is the only thing standing between the factory and its own backlog:
`adw-m9-04` (4 failures) → `adw-m9-06` → the review lane going live, plus
`adw-bug-02`. Every one of those is blocked on a 10-line prompt change.
