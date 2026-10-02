---
id: sabado-50-corroboration-elects-a-scalar-not-recency-edfd1a
type: bug
status: queued
priority: 2
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-47-an-extracted-fact-is-attested-against-the-bytes-the-model-saw-c4e601, sabado-48-a-fabricated-fact-is-caught-by-a-decoy-087211, sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e]
attempts: []
---
# fix(extraction): a scalar is elected by corroboration again, instead of by whichever document is newest

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3, "Two
> correctness bugs", second bullet, and Wave C row 46.
>
> **Depends on 47, 48 and 49 — deliberately.** The 2026-09-12 audit called
> corroboration-first election *"the single largest measured accuracy lever"*.
> Moving the largest lever with no instrument in place is how you get a number
> nobody can defend. The three eval tickets exist so this one can be measured.

## What happens today

```python
CORROBORATED_SCALARS = frozenset({"familleAdulte.conjoint"})
```

**The allowlist has narrowed to one key.** Everything else now elects on
`-recency` — *the newest document wins* — including `statutDOccupation`,
`internetFournisseur` and `creditCapitalInitial`.

Recency is a reasonable tiebreak and a poor elector: a single recent
mis-extraction beats three older agreeing ones.

## The red test

A scalar is extracted three times in agreement from older documents and once,
differently, from the newest. Assert the corroborated value is elected. Today
the newest wins — that is the red.

## Requirements

- [ ] **R1** Corroboration elects, recency breaks ties. The ordering is the
      change; the mechanism already exists.
- [ ] **R2** The allowlist is **widened deliberately, not globally**. Name every
      key added and why, in the PR body. A scalar whose truth genuinely changes
      over time (an address, a provider) may be right to elect on recency —
      **say which you judged that way.**
- [ ] **R3** The change is **measured** against the instrument from `sabado-49`
      before and after. Quote both figures in the PR body. If the number moves
      the wrong way, say so and stop — do not tune until it moves the right way.
- [ ] **R4** No re-read of any mailbox, and no ledger rewrite. Election happens
      at read time; this changes future reads.
- [ ] **R5** `familleAdulte.conjoint` keeps its current behaviour exactly.

## Files

The election module holding `CORROBORATED_SCALARS` · `backend/tests/`.

## Verify

- [ ] The red test above is green.
- [ ] A test asserts recency still breaks a tie between two equally corroborated
      values.
- [ ] Before/after scores from `sabado-49`'s scorer are quoted in the PR body.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Re-reading mailboxes. Backfilling. Any prompt change. Adding a new election
strategy beyond corroboration and recency.
