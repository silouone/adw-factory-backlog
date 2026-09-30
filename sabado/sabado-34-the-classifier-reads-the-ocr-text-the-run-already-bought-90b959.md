---
id: sabado-34-the-classifier-reads-the-ocr-text-the-run-already-bought-90b959
type: bug
status: in-progress
priority: 1
created: 2026-09-30
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: []
---
# fix(extraction): the classifier reads the OCR text the run already paid for, instead of guessing from a page image

> **Finding:** `ai_docs/2026-09-30-sabado-ocr-strategy.md` §2 — *"the finding
> that matters"*. **The largest single quality win available in ingestion, at
> zero new model calls.** The audit ranked it 🥇 of eight candidates.

## What happens today

`classify_document` (`documents.py:598`) takes
`(filename, data, mime, provider, *, hint, vision)`. **It re-extracts the text
layer itself and is never given the OCR text:**

```python
is_pdf = filename.lower().endswith(".pdf") or "pdf" in mime.lower()
text = extract_pdf_text_bytes(data) if is_pdf else ""
by_vision = False
if text.strip():
    parsed = parse_document(text, provider, ...)
else:
    if not vision.take():
        trace.note("document_classification_capped", ...)
        return None
    by_vision = True
```

Two things follow. A non-PDF gets `text = ""` **unconditionally** — and 27 of
42 OCR'd items on a real mailbox were PNG/JPEG. And a document whose OCR
already ran has its markdown sitting in `enrichment_documents.ocr_text`
(`models/enrichment.py:278`) — **on the very row the caller is holding**
(`documents.py:680`, `for index, row in enumerate(pending)`) — and the
classifier never sees it. The run bought the text and threw it away.

Then `CLASSIFY_VISION_MAX_CALLS = 20` (`documents.py:545`) refuses the rest.
Measured on Erwann's read, and arithmetically exact against the class table:

```
documents with NO text layer:                    108
  of which classified (== the by_vision counter): 20   ← exactly the cap
  of which unclassified:                           88   ← classified_at NULL
of the 88 unclassified, filed on a record:          3
```

| Fix | Covers | New model calls |
|---|---|---|
| **Pass `ocr_text` in (this ticket)** | **39 of 88** | **0** |
| Raise the vision cap 20 → 100 | 88 of 88 | 68 |

**This ticket is the free half only.** The cap raise is an operator cost
decision and is explicitly out of scope.

Aggravating, and worth knowing while implementing: the 20 vision
classifications ran on **`ministral-8b-latest`** — `AI_MODEL_TRIAGE` defaults
to `""` (`config.py:74`) and `config.py:66` only verifies that ministral-8b
"handles an image request fine", i.e. does not error. An 8B general **text**
model is guessing at French scans while a dedicated OCR's markdown for the
same document goes unused.

This is also a known-bad configuration independently: building olmOCR, AI2
found image-only prompting *"prone to models completing unfinished sentences,
or to invent larger texts when the image data was ambiguous"*, and their fix
was re-injecting extracted text as layout anchors — a hybrid, not
image-direct (arXiv 2502.18443).

## Requirements

- [ ] **R1** `classify_document` accepts the already-bought OCR text as a
      keyword argument, defaulting to empty so every existing caller is
      byte-identical in behaviour.
- [ ] **R2** The text the classifier uses is: the PDF text layer if non-empty,
      **else the OCR text if non-empty**, else vision. The OCR text is
      consulted **before** `vision.take()` — so a document it can serve does
      not spend a vision call.
- [ ] **R3** The caller passes `row.ocr_text` (`documents.py:680`). It is
      `str | None`; treat `None` and whitespace-only as absent.
- [ ] **R4** The vision budget is only consumed when a vision call is actually
      made. A document served from OCR text must leave `vision` untouched — this
      is the mechanism by which the cap stops being the binding constraint.
- [ ] **R5** `trace` distinguishes the three sources, so the next read's
      artifacts say which path classified each document. Extend the existing
      `trace.note` vocabulary; do not invent a parallel channel.
- [ ] **R6** No new model call, no new provider, no OCR request. This ticket
      spends nothing it did not already spend.

## Files

`backend/app/extraction/documents.py` (`classify_document` at `:598` and its
one call site at `:680`) · `backend/tests/test_documents_filed_on_the_record.py`
or a new module. Nothing else.

## Verify

- [ ] **The red test, first.** A document with **no text layer** and a
      **non-empty `ocr_text`** asserts that it is classified **and** that the
      vision budget was not decremented. It **fails on the current tree** —
      today that document either burns a vision call or returns `None` when the
      cap is spent. That failure is the ticket.
- [ ] A second red test: with the vision budget already exhausted
      (`vision.take()` false), a document carrying `ocr_text` is **still
      classified**. Today it returns `None`.
- [ ] Regression: a document **with** a text layer classifies exactly as before
      and still prefers the text layer over `ocr_text`.
- [ ] Regression: a document with neither still goes to vision and still
      respects the cap.
- [ ] `git diff --stat` touches only the files listed above.
- [ ] Target gates: `mypy app/` · `lint-imports` · `api-debt-check` ·
      `alembic heads` · `alembic-branch-check` · `test` — all green.

## Out of scope

**Raising `ENRICH_CLASSIFY_VISION_MAX_CALLS` (20 → 100) — an operator cost
decision, deliberately held back.** Raising `MAX_PAGES_PER_READ`. Pinning the
OCR model (`sabado-36`). Pricing OCR pages (`sabado-33`). Requesting
`include_blocks` or `confidence_scores`. Changing the triage-tier model, and
changing `AI_MODEL_TRIAGE`'s default — both are real findings and neither is
this ticket.
