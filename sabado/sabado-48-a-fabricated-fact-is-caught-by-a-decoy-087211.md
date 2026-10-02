---
id: sabado-48-a-fabricated-fact-is-caught-by-a-decoy-087211
type: feat
status: queued
priority: 2
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-47-an-extracted-fact-is-attested-against-the-bytes-the-model-saw-c4e601]
attempts: []
---
# feat(extraction): a fabricated fact is caught by a decoy, so the attester is measured instead of trusted

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3 and Wave
> C row 44. The synthesis calls the decoy-salted control **"the best-designed
> single idea in either prototype."**

## What happens today

Nothing measures the *verifier*. An attester that confirms everything passes
every test you would naturally write for it.

The POC solved this: salt the input with **decoys** — fabricated claims that are
not in the source — and measure how many the verifier confirms anyway. It throws
on any decoy retained. That turns "the attester works" from a belief into a
number.

## Requirements

- [ ] **R1** A decoy-salted control over `attest.py`: a corpus run is salted
      with fabricated claims, and the run **fails on any decoy retained**.
- [ ] **R2** It is a **pytest gate**, not a script someone remembers to run.
- [ ] **R3** The decoys are **plausible**, not absurd — a fabricated SIRET, an
      amount off by one digit, a date shifted by a month. A decoy no model would
      ever produce measures nothing. List the decoy families in the PR body.
- [ ] **R4** Decoy generation is **deterministic**. A gate that varies run to
      run is a flake, and this repo already pays for flaky gates elsewhere.
- [ ] **R5** The retained-decoy count is **reported**, not only asserted — the
      number is the instrument's own accuracy and belongs where a human reads it.

## Files

`backend/app/extraction/eval/` · `backend/tests/`. Nothing in the extraction
runtime.

## Verify

- [ ] The gate fails when a deliberately broken attester (one that confirms
      everything) is substituted — demonstrate and say so in the PR body.
- [ ] The gate passes twice in a row with identical output.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Running against a real mailbox. Scoring the extraction prompt, which is
`sabado-49`. Changing `attest.py`'s rules.
