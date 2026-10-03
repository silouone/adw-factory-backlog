---
id: sabado-47-an-extracted-fact-is-attested-against-the-bytes-the-model-saw-c4e601
type: feat
status: in-progress
priority: 2
created: 2026-10-02
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: []
attempts: []
---
# feat(extraction): an extracted fact is re-located in the bytes the model was shown, with no model and no golden set

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3 ("Two
> eval assets exist; neither is in CI") and Wave C row 43.
>
> **This is the first of four tickets that together are the real answer to the
> ingestion question.** The synthesis is explicit: *"the gap is measurement, not
> machinery."*

## What happens today

There is **no in-CI check that an extracted value actually appears in the
document it was extracted from.** Every accuracy signal the project has requires
either a golden set or a model.

The asset already exists, in the wrong repo: `poc/gmail-wave1`'s `attest.mjs` —
**pure, ~190 lines, zero I/O**, a deterministic verbatim relocation of an
extracted value in its source bytes. It verifies **without a model and without a
golden set**. Its French canonicalisation (NFKC → diacritic fold → HTML entities
→ the no-break-space family) is domain knowledge that would be expensive to
rediscover.

It sits in PR #236, which is 568 commits behind and heading for closure.

## Requirements

- [ ] **R1** `attest.mjs` is ported to `backend/app/extraction/eval/attest.py`.
      **Pure: no model, no golden set, no network, no filesystem.** A function
      from (value, source bytes) to a relocation verdict.
- [ ] **R2** The **French canonicalisation is carried verbatim in behaviour** —
      NFKC, diacritic fold, HTML entities, the no-break-space family. Port the
      *rules*, not the JavaScript. Each rule gets a test naming the real French
      input that motivated it.
- [ ] **R3** The port is **test-first and behaviour-pinned**: before porting,
      write the cases from the `.mjs` as Python tests. A rule you cannot state a
      case for is a rule you do not port — say which, if any, in the PR body.
- [ ] **R4** It is callable from a test and from a script, and it is **not
      wired into the extraction runtime**. This ticket adds an instrument, not a
      gate.
- [ ] **R5** `UNVERIFIED:` the synthesis records that the POC's own golden set
      is *"22 facts, all byte-identical to a prior run's output, zero checked
      against an original document"* — a regression baseline, not ground truth.
      **Do not port it.** Port the attester only.

## Files

`backend/app/extraction/eval/attest.py` (new) · `backend/tests/`. Nothing in the
extraction runtime.

## Verify

- [ ] A test asserts a value present in the source under a different
      normalisation still relocates (accents, NBSP, entity).
- [ ] A test asserts a value absent from the source does **not** relocate.
- [ ] The module imports nothing from `app/extraction/` outside `eval/` —
      `lint-imports` should be able to see that.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Running it over any real mailbox. The decoy control, which is
`sabado-48-a-fabricated-fact-is-caught-by-a-decoy-087211`. Any golden set.
