---
id: adw-bug-01-capture-parser-envelope
type: bug
status: done
priority: 1
created: 2026-09-14
depends: []
attempts: [{"runId":"adw-bug-01-capture-parser-envelope-1789401393794","branch":"adw/adw-bug-01-capture-parser-envelope","workspace":"/Users/silouane/adw-factory/runs/adw-bug-01-capture-parser-envelope-1789401393794/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/40","provider":"claude","model":"sonnet"}]
---
# The capture parser reads a shape that has never been on disk — and the metrics rail turns that silence into "100% model time"

> Found 2026-09-14 while wiring tool-call dots into the grid prototype
> (`src/web/GRID-PROTOTYPE-NOTES.md`). **Two defects, one root, and the second
> survives fixing the first** — so this ticket fixes both or it is not done.

## 1. Defect A — `parseCaptureLine` cannot parse a single real line

`src/web/capture.ts` expects each capture line to be the hook payload flat:

```json
{"session_id": "…", "hook_event_name": "PreToolUse", "timestamp": 1789392134617}
```

Every capture file the factory has ever written wraps it in a cLens envelope:

```json
{"t": 1789392134617, "event": "PostToolUse", "sid": "d27af066-…",
 "data": {"session_id": "…", "hook_event_name": "PostToolUse",
          "tool_name": "Edit", "duration_ms": 431, …}}
```

All three fields `parseCaptureLine` requires — `session_id`, `hook_event_name`,
`timestamp` — are absent at the top level. It returns `undefined` for **every
line of every file**.

Measured on `runs/adw-fe-13-run-screen-design-rework-1789391893971/`:

| | |
|---|---|
| session files named by the journal | 3, all `capture.ok: true` |
| lines across them | 342 |
| `PreToolUse` + `PostToolUse` records | 171 + 165 |
| **records `parseCaptureLine` accepts** | **0** |

## 2. Defect B — the metrics rail reports a confident wrong number

This is the dangerous half, and it is **independent of Defect A**.

`readSessionCaptures` reads the file *successfully* — it is valid JSONL — so
the session lands `{status: "ok", dots: []}`, not `"none"`. `metrics.ts` only
withholds `time` when the status is `"none"`. With zero records it therefore
computes `toolMs: 0` and `modelMs: durationMs`:

```
plan   time: {toolMs:0, subagentBlockedMs:0, modelMs:328421}
build  time: {toolMs:0, subagentBlockedMs:0, modelMs:1872323}
test   time: {toolMs:0, subagentBlockedMs:0, modelMs:919781}
```

**100% model time, 0% tool time, on every node of every run.** Not an absence —
an assertion, rendered as the run screen's headline split bar.

Fixing the parser does not close this. A session whose capture is `ok` with
genuinely zero `PostToolUse` records (a node that made no tool calls, or a
capture truncated before any landed) still reports 100% model time. The
codebase already distinguishes these two states everywhere else — `capture.ts`
is explicit that a missing capture and a captured-but-silent session must not
look the same. `metrics.ts` collapses them.

## 3. Blast radius — what this silently broke, and for how long

| ticket | what it claims | what actually renders |
|---|---|---|
| `adw-fe-07-tool-dots` | a dot per tool call along each block | **nothing, on any real run** |
| `adw-fe-10-metrics-rail` | the model/tool time split | a fabricated 100/0 |
| `adw-fe-13` §5 | *"model vs tool colour is the page's thesis"* | a solid orange bar |

`adw-fe-13`'s ticket, and the `PROTOTYPE-NOTES.md` it was built from, both
quote **"~90% model time"** as the measurement that motivated the whole colour
scheme. That figure must be re-derived once this lands — it may be an artifact
of Defect B. (Re-deriving it is `adw-fe-15`'s job, not this ticket's.)

## 4. Why the suite is green — read this before writing the test

`test/web/capture.test.ts` hand-builds every fixture in the **flat** shape
(`hook_event_name` at the top level). The fixtures agree with the parser.
Neither agrees with the disk. **No test in the repo reads a real capture
file**, which is exactly how a parser can be 100% wrong and 100% green.

That is the trap to avoid repeating: a regression test built from another
hand-written fixture would pass against the current code too.

## 5. Red test — Article I, and it must fail for the right reason

- [ ] **The regression test uses a REAL captured line**, copied verbatim from a
      banked `runs/<id>/.clens/sessions/*.jsonl` into the test file as a
      literal. Not hand-authored, not reshaped. A test that invents its own
      envelope proves nothing about what the factory writes.
- [ ] **A second red test for Defect B**: a session that reads `ok` with zero
      `PostToolUse` records must NOT yield `time: {toolMs: 0, modelMs: dur}`.
- [ ] Both red before any fix. `red-check` classifies them.

## 6. Requirements

- [ ] `parseCaptureLine` accepts the enveloped shape: `sid`/`t` at the top
      level, the hook payload under `data`. Timestamp from `t`; `tool_name`
      and `duration_ms` from `data`.
- [ ] **Decide and state whether the flat shape stays supported.** The existing
      tests depend on it. Either keep both (and say why in a comment — e.g. an
      older capture format still on disk), or convert the fixtures and drop it.
      Silently supporting a shape nothing writes is how this bug hid.
- [ ] Defect B: a capture that parsed cleanly but yielded **no `PostToolUse`
      records at all** must withhold `time`, exactly as `status: "none"` does.
      A node with tool calls whose durations sum to zero is a different case
      and keeps its `{toolMs: 0, …}`.
- [ ] No renderer changes here. `render-run.ts` already draws ticks and the
      split bar; this makes them true.

## Verify

- [ ] Against a banked run with captures, `loadRunDetail` yields non-zero
      `capture.dots` and a `time` split whose `toolMs > 0`.
- [ ] A run whose journal records `capture.ok: false` still yields
      `status: "none"` and no `time` — unchanged.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

Rendering, wording and layout on either screen — `adw-fe-14` (grid) and
`adw-fe-15` (run screen) own those, and both depend on this landing first.
