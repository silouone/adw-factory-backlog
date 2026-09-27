---
id: adw-m9-01-review-verdict-contract
type: feat
status: done
priority: 1
created: 2026-09-12
epic: adw-m9
depends: []
attempts: [{"runId":"adw-m9-01-review-verdict-contract-1789218132695","branch":"adw/adw-m9-01-review-verdict-contract","workspace":"/Users/silouane/adw-factory/runs/adw-m9-01-review-verdict-contract-1789218132695/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/10","provider":"claude","model":"sonnet"}]
---
# The review verdict — typed findings, pure parser

## Context

`specs/adw-v1.3-review-lane.md` R3. A reviewer that returns prose cannot drive
a routing decision: `adw-m9-04` must decide, in code, whether a finding sends
work back. So the verdict is a TYPE before it is a prompt.

This is roadmap item 2.1 (typed agent envelopes) applied to exactly one
envelope, rather than as a speculative framework. Pure module: no I/O, no
clock, no agent (Art. III, Art. IX).

## Deliverables

`src/pipeline/review/verdict.ts` and its test.

## Requirements

- [ ] A `Finding` type carrying: `axis` (`"standards" | "spec"`), `severity`
      (`"hard" | "judgement"`), `file`, `line` (optional — some findings are
      file-level), `citation` (the documented standard, or the ticket line, the
      finding is measured against), and `summary`.
- [ ] A `SpecFinding` discriminant for the spec axis' three questions:
      `"missing"` (asked for, absent or partial), `"unasked"` (scope creep),
      `"wrong"` (looks implemented, implementation looks wrong). The spec axis'
      routing (adw-m9-04) keys on `"missing"`, so it must be a TYPE not a
      keyword in prose.
- [ ] A `ReviewVerdict` = `{ axis, findings: readonly Finding[] }`. One verdict
      PER AXIS — never a merged object. R2 forbids merging, and a combined type
      would invite it.
- [ ] `parseVerdict(raw: string): ParseResult` — a pure parser from the agent's
      final message to a validated verdict, or a deterministic list of EVERY
      structural problem (the `targets/loader.ts` and `intake/ticket.ts`
      precedent: report all problems at once, never one per re-prompt).
- [ ] An empty `findings` array is VALID and means "no findings" — distinct
      from a parse failure. A clean review must be expressible.
- [ ] `isBlocking(finding): boolean` — the D1 floor as a PURE PREDICATE:
      `true` for a standards-axis `severity: "hard"`, and for a spec-axis
      `kind: "missing"`. Everything else is `false`. In code, not in the
      reviewer's judgement (Art. III).

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun run test`
- [ ] Red tests first, covering: each `isBlocking` case both ways; an empty
      findings array parsing OK; a malformed verdict reporting every problem in
      one pass, not the first.
- [ ] No import of `node:fs`, `node:child_process`, or the agent SDK anywhere
      in `verdict.ts` — the module is pure.

## Out of scope

The node that produces a verdict (adw-m9-02), the prompts (adw-m9-03), routing
on it (adw-m9-04). Generalising typed envelopes to `plan`/`test` (roadmap 2.1
proper) — this ticket types ONE envelope.
