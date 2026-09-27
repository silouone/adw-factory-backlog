---
id: adw-bug-05-concurrency-ceiling-stalls-the-stream
type: bug
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-bug-05-concurrency-ceiling-stalls-the-stream-1789427476281","branch":"adw/adw-bug-05-concurrency-ceiling-stalls-the-stream","workspace":"/Users/silouane/adw-factory/runs/adw-bug-05-concurrency-ceiling-stalls-the-stream-1789427476281/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/47","provider":"claude","model":"sonnet"}]
---
# "Response stalled mid-stream" is a ~300-second idle-stream timeout, and our longest-thinking nodes walk into it by design

> **Third revision of this ticket, and the first with a mechanism that holds.**
> Draft 1 blamed subscription throttling — the operator refused it (they run
> 10+ concurrent Claude sessions daily) and was right. Draft 2 blamed CPU
> contention from parallel test suites — plausible, and also wrong. The
> evidence below is from the cLens captures and is decisive.
>
> The operator's framing is the correct one: *"2 builds should be FINE,
> otherwise what's the point of having a software factory if we can't send A
> LOT through."* Concurrency is not the defect. This is.

## 1. The measurement

For every agent node, the gap between its **last tool call** and its
`node-end` — i.e. how long the stream carried no tool activity before it ended:

| | n | median trailing silence |
|---|---|---|
| **successful** agent nodes | 142 | **19s** |
| **stalled** agent nodes | 8 | **~305s** |

The stalled ones, individually:

```
714.9s  plan    adw-bug-02-run-screen-axis-and-liveness
338.6s  plan    adw-m9-04-review-fix-loop
307.6s  plan    adw-m9-04-review-fix-loop
307.4s  plan    adw-bug-02-run-screen-axis-and-liveness
304.5s  plan    adw-m9-04-review-fix-loop
303.0s  build   adw-fe-14-grid-screen-console-design
302.6s  plan    adw-bug-02-run-screen-axis-and-liveness
302.3s  plan    adw-m9-04-review-fix-loop
```

**Seven of eight land between 302.3s and 338.6s. Four of them inside a 5.3
second window.** That is not contention and it is not load — that is a fixed
**~300-second timeout on a stream that has gone idle**, with a little jitter.

Nothing in this repo sets a 300s timeout. It is the SDK's or the network path's.

## 2. Why it is our longest nodes, and why by design

The tool traces show the tools are not slow — `Read` and `Grep` return in
0.0–0.5s. What is slow is the **model time between tool calls**, and then a
single long final generation with no tool calls at all.

`prompts/feature-plan.md` asks for exactly that shape:

> Your final message **is** the plan artifact: it is captured verbatim and
> handed to the build agent as its plan. Make it self-contained and specific…
> Do not run git at all, and make no changes to the working tree: **the plan
> is your only output**.

So `plan` is *instructed* to stop calling tools and emit one long,
self-contained document. During that generation the stream is idle from the
transport's point of view. Exceed ~300s and it is killed.

`build` hits the same wall for the same reason — its final build report is
long. `adw-fe-14`'s build made **258 tool calls**, finished all its work, then
went quiet for **303.0s** composing its report and died four deterministic
nodes short of a PR. The work was complete and green; only the report was lost.

**This is why every stall is on `plan` or the end of `build`, and never on
`test` or a gate node.** Those keep touching tools.

## 3. This finally explains the concurrency correlation

Draft 2 was not wrong that stalls cluster at concurrency ≥2 — they do. But
concurrency is not the cause, it is the **push over a fixed cliff**: under
load, token delivery slows, so a final generation that normally lands in 200s
takes 350s and crosses 300s.

Fix the cliff and the concurrency limit largely evaporates — which is the
answer to *"2 builds should be FINE"*. They should. A concurrency bound would
have treated the symptom and capped the factory's whole reason for existing.

## 4. The fix — make the long artifact a tool call, not a final message

- [ ] **`plan` writes its plan to a file** (`Write`), and the node reads that
      file. Three things fall out of one change:
      - every write is a tool call, so the stream never goes idle for 300s;
      - the artifact is **durable**, so a stall after the plan exists no longer
        loses it (today it is recoverable only by hand out of
        `runs/<id>/prompts/0003-build.txt` — see
        `ai_docs/2026-09-14-m9-04-recovered-plan.md`);
      - `plan` stops being the only node whose output lives in a message.
- [ ] **`build` writes its report to a file** for the same reason. Its 303s
      report loss cost a complete, green, 117-test feature.
- [ ] **Chunk, don't just relocate.** One `Write` of a 10K-token document is
      still one long generation. The prompt should ask for the artifact to be
      built up in sections, so no single generation approaches the ceiling.
- [ ] Check whether the SDK exposes a stream/idle timeout we can raise. If it
      does, raise it **as well** — but not instead. A 300s ceiling we do not
      control is not something to design against.

## 5. And journal the reason — this cost hours of diagnosis

A stalled `node-end` carries no reason at all:

```json
{"type":"node-end","node":"plan","outcome":"retry",
 "details":{"kind":"agent","usage":{"tokens":0,"turns":0,…}}}
```

*"API Error: Response stalled mid-stream"* is printed to the terminal and then
lost. This diagnosis required inferring stalls from `turns == 0` and
reconstructing trailing silence from capture timestamps. Article VI wants "why
did it stop" answerable from artifacts alone.

- [ ] Journal the classified reason on a `retry`/`fail` node-end: the
      provider's message, and the trailing-silence duration.
- [ ] Red test: a fake query returning a stalled result → the journal carries
      the reason, greppable, without reading the terminal.

## 6. Retrying is the wrong response and makes it worse

`adw-resilience-01`'s transient-retry fires on this and re-runs `plan` from
scratch — a node that takes 11+ minutes to reach the same cliff. Three
attempts is **34 minutes to arrive back where it started**, which is what
`adw-m9-04` and `adw-bug-02` each did, twice.

- [ ] Raising `TRANSIENT_RETRY_MAX_ROUNDS` is **not** the fix — it buys 45
      minutes of failure instead of 34. Once §4 lands, verify the retry stops
      firing at all rather than tuning its ceiling.

## Verify

- [ ] `adw-m9-04` and `adw-bug-02` — the two tickets this has blocked
      repeatedly — both complete.
- [ ] Median trailing silence on agent nodes stays near the successful-node
      figure (19s), measured from captures.
- [ ] Two concurrent feat runs both complete. That is the acceptance bar the
      operator actually asked for.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
