---
id: sabado-33-an-ocr-page-reaches-the-cost-meter-d1670e
type: bug
status: queued
priority: 2
created: 2026-09-30
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: []
---
# fix(extraction): an OCR page is priced, so a run stops under-reporting its own bill

> **Finding:** `ai_docs/2026-09-30-sabado-ocr-strategy.md` §2. This is the
> **same defect `pricing.py`'s own docstring was written about**, one file
> over — and the file already calls itself "that one-line fix".
>
> **Shares `cost.py` with `sabado-32`,** which owns the `STAGE_*` constant
> block (~`:60-75`). This ticket owns the pricing / partial-price region
> (~`:660-690`).

## What happens today

`pricing.py` prices models. Its `MODEL_PRICES` table on `origin/main` has
exactly three rows:

```python
MODEL_PRICES: "dict[str, ModelPrice]" = {
    "ministral-8b-latest":   ModelPrice(input_usd=0.15, output_usd=0.15),
    "mistral-medium-latest": ModelPrice(input_usd=1.50, output_usd=7.50),
    "mistral-large-latest":  ModelPrice(input_usd=0.50, output_usd=1.50),
}
```

**There is no OCR row, and OCR is not billed as a *model* — it is billed per
page.** So `price_run` is never asked about it, `price_is_partial` stays
`False` (`cost.py:678-680`), and the run reports a total it presents as
complete. Measured on Erwann's read: **reported $3.55, true bill $4.15 — short
by 17 %.**

The file's own docstring is the argument for this ticket:

> "The medium row was missing, and the gap was not academic: read B of
> 2026-09-07 ran on `mistral-medium-latest`, reported **$0.068** and had in
> fact spent **$1.75** — a run under-reporting its own bill by 25×.
> `price_run` prices an unknown model at `None` rather than at zero precisely
> so this shows up, and this line is that one-line fix."

The same protection was never extended to the one cost line that is not a
model. Pages billed are already known — `ocr.py` meters them against
`MAX_PAGES_PER_READ` — so nothing new has to be measured, only priced.

Volume, for sizing: **407 pages across four real mailboxes, $1.63 lifetime.**
The money is trivial; a total that silently omits a line item is not.

## Requirements

- [ ] **R1** OCR pages are priced. Current public rate **$4.00 / 1 000 pages**
      → `$0.004/page`. Carry the rate the way `MODEL_PRICES` carries its own —
      a dated, named constant, not a literal at the call site.
- [ ] **R2** The OCR line appears in the run's cost snapshot as its own entry,
      distinguishable from model spend. Do not fold it into a model's total.
- [ ] **R3** `price_is_partial` is `True` when OCR pages were billed and the
      OCR rate is absent or unknown — the same posture `price_run` takes for
      an unpriced model. **An unpriced page must not read as a free page.**
- [ ] **R4** A run that billed zero OCR pages reports exactly what it reports
      today. No change to any existing figure.
- [ ] **R5** If the rate is sourced from configuration rather than code, an
      unset value is `None` and partial — never `0.0`.

## Files

`backend/app/extraction/pricing.py` ·
`backend/app/extraction/cost.py` (pricing / partial region only) ·
`backend/tests/` — the existing pricing test module, or a new one. Nothing else.

## Verify

- [ ] **The red test, first.** A run whose meter recorded N billed OCR pages
      asserts the priced total includes `N × rate`. It **fails on the current
      tree** — the OCR spend is simply absent. That failure is the ticket.
- [ ] A second red test: with the OCR rate unknown, `price_is_partial` is
      `True`. Today it is `False`.
- [ ] Regression: a zero-OCR run's total is byte-identical before and after.
      Show it in the PR body.
- [ ] Reproduce the audit's arithmetic in the PR body: 150 pages × $0.004 =
      $0.60, and state the reported-vs-true figures it closes.
- [ ] Target gates: `mypy app/` · `lint-imports` · `api-debt-check` ·
      `alembic heads` · `alembic-branch-check` · `test` — all green.

## Out of scope

Raising `MAX_PAGES_PER_READ`. Requesting `include_blocks` or
`confidence_scores`. Changing OCR vendor or model — `sabado-36` owns the pin.
Any chat-side metering: `provider.chat()` lives on the `feat/chat-calls-tools`
branch and is not reachable from `main`.
