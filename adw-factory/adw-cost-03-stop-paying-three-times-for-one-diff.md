---
id: adw-cost-03-stop-paying-three-times-for-one-diff
type: feat
status: done
priority: 2
review: true
created: 2026-09-21
caps: {minutes: 150, turns: 700}
depends: [adw-cost-01-bound-what-a-tool-call-may-inject]
attempts: [{"runId":"adw-cost-03-stop-paying-three-times-for-one-diff-1790031865585","branch":"adw/adw-cost-03-stop-paying-three-times-for-one-diff","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-cost-03-stop-paying-three-times-for-one-diff-1790031865585/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/109","provider":"claude","model":"sonnet"}]
---
# P3 — the review family's cost is its prompt, and the diff is sent three times

> **Evidence:** `ai_docs/2026-09-21-token-audit-FULL.md` §3, §6 Cluster D.
> **Modelled saving: ~138M — 4.5%.** Smaller than `adw-cost-01`/`02`, and
> deliberately separate from them, because **the review family's cost
> mechanism is the opposite one.**

## Why this is not part of `adw-cost-01`

`adw-cost-01` exists because for `build`/`test`/`plan` the prompt is noise —
96–99.5% of their context is accumulated tool output. **For the review family
the prompt IS the cost:**

| node | prompt on disk | context/turn | prompt as % of context |
|---|---|---|---|
| `review-spec` | 27K–**336K B** ≈ 7–**83K tok** | 101,384 | **7–82%** |
| `review-fix` | 28K–179K B ≈ 6–44K tok | 126,501 | 5–35% |
| `review-standards` | 26K B ≈ 6K tok | 95,865 | ~7% |

One measured `review-spec` prompt is **335,728 bytes ≈ 83K tokens**. Applying
`adw-cost-01`'s tool-output digest to these nodes would do almost nothing —
the mass is in the assembled prompt, and it is there **three times per round**
because each axis is its own session over the same diff.

Observed growth within one run (`adw-render-05`): round-1 prompts were
27–30 KB; after round 1's deletions the round-2 prompts were **84–91 KB** —
the diff quadrupled because a deletion diff carries every removed line, and
that 76 KB was then re-sent to all three nodes.

## Requirements

- [ ] **R1 — one session, three verdicts.** `review-standards`, `review-spec`
      and the in-process standards recheck load the diff **once** and emit
      separately-addressed verdicts, instead of three sessions each paying a
      full load and then re-reading it every turn.
- [ ] **R1a — the verdict contract is unchanged.** `unionBlockingFindings` /
      `unionRoutingFindings` keep receiving one verdict per axis with the same
      shape. `adw-review-01`'s routing floor and `adw-review-02`'s unresolved-
      findings-into-the-PR path must be byte-identical in behaviour.
- [ ] **R1b — the axes stay independent in judgement.** Fusing the *session*
      must not let the spec axis see the standards axis's reasoning and
      anchor on it. If one prompt cannot produce two genuinely independent
      verdicts, **say so and fall back to R2 alone** — the audit's own
      perspective-diversity argument outranks a 4.5% saving.
- [ ] **R2 — send a hunk manifest, not the whole diff.** Reviewers receive
      changed files, hunk headers and pointers, and pull only the hunks they
      need. A deletion-heavy sweep must not cost 83K tokens of `-` lines to
      review.
- [ ] **R3 — content-address the hunks across rounds.** Round N+1 sends only
      hunks whose hash changed since that reviewer last saw them, plus a map
      of unchanged hunks to their prior verdict. `adw-review-03` (#97) already
      threads the prior round's findings; this is the same idea for the diff.
- [ ] **R4 — no change to what a review says.** `adw-v1.3`'s routing floor
      and severity semantics are untouched. This is transport.

## Verify

- [ ] Red test first (Art. I): assert the diff is currently assembled into
      three separate prompts per round. RED until R1.
- [ ] Red test: a fused review emits two verdicts whose findings sets are
      identical to the two-session baseline on a fixture diff.
- [ ] Red test for R1b: a fixture where the standards axis is clean and the
      spec axis is not (and vice versa) still routes exactly as today.
- [ ] Red test for R3: an unchanged hunk is not re-sent in round 2, and its
      prior verdict is still visible to the reviewer.
- [ ] **Measured:** review-family cacheRead and assembled prompt bytes before
      and after, on a real run. Baseline to beat: **251M cacheRead**, and a
      worst-observed single prompt of **335,728 bytes**.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- The review loop's convergence — `adw-review-03` (#97) fixed it, and **every
  `review-spec` routing death in the audit window predates #88**. This ticket
  is about what a review *costs*, not whether it terminates.
- Removing an axis. Two axes is a spec decision (`adw-v1.3`), not a cost one.
