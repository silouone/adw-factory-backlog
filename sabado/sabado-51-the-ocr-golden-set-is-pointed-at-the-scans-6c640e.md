---
id: sabado-51-the-ocr-golden-set-is-pointed-at-the-scans-6c640e
type: feat
status: queued
priority: 3
created: 2026-10-02
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e]
attempts: []
---
# feat(extraction): the OCR golden set is pointed at the scans, because no published benchmark reads French admin

> **Finding:** `ai_docs/2026-09-30-sabado-ocr-strategy.md`, surfaced in
> `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §4 and Wave D row 48.
>
> **Depends on `sabado-49`** — it vendors `lab/score.py`, which this reuses. Two
> copies of the scorer is the drift this repo's contracts exist to prevent.
>
> **This is not "build an eval". It is "point the eval you have at the scans."**

## What happens today

`keys/quentin-ocr.csv` — **61 rows, 54 valued**, hand-read against
`corpus/quentin-ocr.json` (271 messages, the only corpus carrying bodies) —
already exists, with a replayable baseline (`20260908-020024-ocr-quentin-ocr`)
and `lab/score.py`. It is not pointed at anything.

It is **load-bearing**, and more so than when it was written:

- Vendor and third-party benchmark numbers for the incumbent **directly
  conflict**. Mistral's launch page claims olmOCR-bench 85.20 / OmniDocBench
  93.07; the live third-party OmniDocBench v1.7 leaderboard puts Mistral OCR at
  **85.66 with seven models above it**, and olmOCR-bench at **72.0, below
  Marker's 76.0**. Mistral's own page hedges that competitor scores are internal
  reproductions.
- **No published benchmark contains French.** OmniDocBench covers `en`,
  `simplified_chinese`, `en_ch_mixed` and two table variants. SABADO reads *avis
  d'imposition*, *taxe foncière*, *actes notariés*, *attestations Ameli*.
- Mistral itself says: evaluate on your own documents.

> **This is the only obtainable French-admin accuracy evidence.**

## Requirements

- [ ] **R1** `keys/quentin-ocr.csv` and `corpus/quentin-ocr.json` are vendored
      beside `sabado-49`'s assets and scored with **the same `lab/score.py`** —
      not a copy.
- [ ] **R2** `20260908-020024-ocr-quentin-ocr` is replayed as the **control**,
      and the run reproduces it. If it does not reproduce, **stop and report the
      delta** — a baseline that no longer reproduces is a finding, not an
      obstacle to work around.
- [ ] **R3** The result is a **number a human reads**, recorded where the next
      OCR decision will look for it. State it in the PR body.
- [ ] **R4** It is runnable on demand and **does not gate every PR** — it reads
      scans and may cost calls. Say what one run costs.
- [ ] **R5** **`UNVERIFIED:` the contested vendor/third-party figures above are
      reproduced from the OCR strategy pass and are NOT re-verified by this
      ticket.** Do not quote them as results; they are the reason the ticket
      exists.

## Files

`backend/app/extraction/eval/` · `backend/tests/`. Nothing in the extraction
runtime.

## Verify

- [ ] The control replays, or the delta is reported.
- [ ] The scorer is imported, not copied — `git grep -c 'def score'` across
      `eval/` is 1.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

**Switching OCR vendor** (saves $0.26 per household; no French evidence supports
it) and **self-hosting any OCR model** (break-even ≈1 520 new households/month
against four households ever; no GPU in the deployment). Both are recorded
refusals, with numbers, in the OCR strategy pass — do not re-propose them.
