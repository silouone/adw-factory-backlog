---
id: adw-bug-12-a-reviewer-formatting-slip-kills-a-green-run-and-leaves-no-evidence
type: bug
status: done
priority: 1
review: false
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-bug-12-a-reviewer-formatting-slip-kills-a-green-run-and-leaves-no-evidence-1789684235106","branch":"adw/adw-bug-12-a-reviewer-formatting-slip-kills-a-green-run-and-leaves-no-evidence-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-12-a-reviewer-formatting-slip-kills-a-green-run-and-leaves-no-evidence-1789684235106/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/73","provider":"claude","model":"sonnet"}]
---
# The first review the factory ever ran killed a green run over output formatting, and threw away the evidence

> Review is default-ON for `chore`, `bug` and `feat` (`adw-m9-06-lane-wiring`).
> Its first live execution blocked a run whose gates were green and whose fix
> was correct. Until this is fixed, every dispatched ticket can be killed by a
> reviewer's formatting slip, and you cannot see what it said.

## Evidence — one run, four separate defects

Run `adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run-1789673356704`.
Chain reached `gates` green (2175 tests, lint + tsc clean, fix verified
correct), then:

```
■ run-end blocked
node "review-standards" axis "standards": verdict parse failed
  — invalid JSON: JSON Parse error: Unexpected identifier "This"
```

The reviewer emitted prose beginning with `This` instead of raw JSON. The
prompt is emphatic — `prompts/review-standards.md`: *"Respond with **ONLY** raw
JSON. No markdown code fence… Any wrapping text breaks the parser."* It ignored
it. That is the trigger. The four defects below are why a trigger became a
blocked run with nothing to debug.

**D1 — the raw response is discarded.** All that survives anywhere is
`Unexpected identifier "This"`. Not in `journal.jsonl`, not in `spans/`, not in
`artifacts/`. You cannot distinguish "wrote prose", "wrote fenced JSON", and
"refused to answer" — three different bugs with three different fixes. This
contradicts `adw-m9-06`'s own R7/Art. VI bar: *"what did the reviewer object
to, and what happened next, must be answerable from artifacts alone."* Every
other agent node persists an artifact. The blocked run's
`runs/<id>/artifacts/` holds `plan.md`, `build.md`, `build-test-only.md`,
`test.md` — and no review artifact, because review writes none.

**D2 — a parse failure is unretryable and terminal.**
`src/pipeline/review/fix-loop.ts:17` documents it: *"review-standards (plain
adw-m9-02 node, UNCHANGED, no .retry)"*. Only `review-spec` carries
`.retry`, and that routes **findings**, not malformed output. So `fail` blocks
the run immediately. A malformed verdict is the most retryable error the
factory has — the remedy is to ask again.

**D3 — the parser has zero tolerance.** `src/pipeline/review/verdict.ts:84` is
a bare `JSON.parse(raw)`. No code-fence stripping, no outermost-object
extraction. A hard parser sits behind a soft prompt instruction; models wrap
JSON in ` ```json ` fences routinely.

**D4 — no test ever exercised a malformed shape.** Nothing in
`test/pipeline/review/` or `test/pipeline/nodes/review.test.ts` feeds fenced or
prose-wrapped output; every fixture is already-clean JSON. The one failure mode
that occurred on first contact with production is the one never tested.

## The structural remark

`src/pipeline/nodes/review.ts:302` reads
`streamResult.patch?.[outputKey]` — the agent's **final message**, the same
channel the build nodes use. But the build prompts deliberately do not trust
it: *"**WRITE it to `.adw/artifacts/build.md`.** That file — not your final
message…"* (`prompts/feature-build.md`, `chore-build.md`, `bug-build-test.md`).
Review asks for machine-parseable output through the one channel the rest of
the factory already learned to route around.

## Requirements

- [ ] **R1 — persist the raw verdict BEFORE parsing** (fixes D1). Write it to
      `.adw/artifacts/review-<axis>.md` in the workspace. **No new plumbing:**
      `src/observability/artifact-sink.ts` already copies every `.md` under the
      workspace artifacts dir into `runs/<id>/artifacts/` at run end, and was
      built precisely for blocked runs — *"a `blocked` run never opens a PR, so
      for exactly the runs whose report matters most, that workspace copy was
      the ONLY copy."* The write must happen even when parsing then fails —
      that is the whole point.
- [ ] **R2 — a parse failure is retryable, bounded** (fixes D2). Re-prompt the
      SAME axis, quoting the malformed output back, with a low ceiling (1–2
      rounds — the operator can tune from journal data; do not raise the
      run-wide caps here). Exhausting it still blocks, naming the ceiling, with
      every raw attempt persisted per R1. This is distinct from
      `review-spec`'s findings-routing retry and must not be confused with it.
- [ ] **R3 — tolerant extraction, strict validation** (fixes D3). Strip a
      leading/trailing markdown fence; extract the outermost balanced `{…}`
      from surrounding prose. Then validate EXACTLY as strictly as today —
      `parseVerdict`'s field checks, unknown-field rejection and axis matching
      are unchanged. Tolerance applies to the *envelope*, never the contract.
- [ ] **R4 — journal the failure with a diagnosable shape.** The `node-end`
      details must say which failure occurred (unparseable / parsed-but-invalid
      / empty response) and point at the persisted artifact — not just echo a
      `JSON.parse` message.
- [ ] **R5 — a reviewer failure must never be more fatal than a reviewer
      finding.** A blocking *finding* routes to a bounded fix loop and only then
      blocks. A malformed *verdict* currently blocks instantly, so the reviewer
      is harshest when it is least intelligible. After R2 the two paths are
      comparably bounded.
- [ ] **Unchanged:** the S2.7 push guard (a blocked review still never reaches
      `push`); the strict verdict contract; `review: false` opt-out; the
      prompts' "ONLY raw JSON" instruction (tolerance is a backstop, not a
      licence to loosen the ask).

## TDD (Art. I — non-negotiable)

Red first, against the REAL `makeReviewNode`/`makeReviewGateLoop` seam with a
fake query — the same seam `adw-bug-10` used, the only layer where this is a
genuine reproduction rather than a proxy.

- [ ] A fake query returning ``` ```json\n{...}\n``` ``` — RED today, green
      after R3.
- [ ] A fake query returning `This diff looks clean. {"axis":"standards",
      "findings":[]}` — prose preamble, the exact observed failure. RED today.
- [ ] A fake query returning unparseable prose with no JSON at all: the raw
      text is persisted (assert on the artifact write), the axis is retried,
      and after the ceiling the run blocks naming it. RED today on all three.
- [ ] A fake query returning valid JSON with an invalid FIELD (e.g.
      `severity:"critical"`) is NOT retried as a formatting slip — it is a
      contract violation and must stay distinguishable from D3's envelope case.
- [ ] Regression: a clean-JSON verdict behaves byte-identically to today, and
      a blocking finding still routes to `review-fix` and still cannot reach
      `push`.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] Re-run the salvaged `adw-bug-11` change through a live lane with review
      enabled and reach `open-pr` — the run this ticket exists because of.

## Out of scope

Forcing the verdict through a structured-output / tool-call channel instead of
free text. That would remove this failure mode rather than mitigate it and is
the better long-term answer, but it changes the agent-node contract for one
node type and deserves its own ticket and spec amendment — file it as a
follow-up rather than smuggling it in behind a bug fix. Also out: changing what
the reviewer judges, the two-axis split, and `review-fix`'s own loop.
