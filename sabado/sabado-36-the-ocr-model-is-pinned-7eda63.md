---
id: sabado-36-the-ocr-model-is-pinned-7eda63
type: chore
status: in-progress
priority: 3
created: 2026-09-30
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-36-the-ocr-model-is-pinned-7eda63-1790770731283","branch":"adw/sabado-36-the-ocr-model-is-pinned-7eda63","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-36-the-ocr-model-is-pinned-7eda63-1790770731283/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1323","provider":"claude","model":"claude-sonnet-5-5"}]
---
# chore(extraction): the OCR model is pinned by date, as the repo's own doctrine requires

> **Finding:** `ai_docs/2026-09-30-sabado-ocr-strategy.md` §2, third bullet.
> Smallest ticket in the wave. Touches one file and one line.

## What happens today

`backend/app/extraction/ocr.py`, on `origin/main`:

```python
#: `mistral-ocr-4-1`, confirmed present on our key via ``GET /v1/models``.
MODEL = "mistral-ocr-latest"
```

The comment names the dated model. The code uses the floating alias. So the
comment is a claim about a version the code does not request, and **a
vendor-side swap behind `-latest` changes behaviour and price silently** —
with `sabado-33` landing, it would change a figure the run now reports as
complete.

This contradicts the repo's own stated rule, in `pricing.py`'s docstring:
"the deployed secrets pin the dated model name behind each `-latest` alias".
Every model lane follows it. The OCR lane does not.

## Requirements

- [ ] **R1** The OCR model is requested by its dated name. Follow whatever
      pattern the model lanes already use to pin — read `model_call.tier_model`
      and the `AI_MODEL_*` settings first and match them; do **not** invent a
      second mechanism.
- [ ] **R2** If the pin is environment-driven, the shipped default is the dated
      name and `deploy/secrets.env.example` documents it beside its siblings.
      An unset value must not silently fall back to `-latest`.
- [ ] **R3** The comment at `ocr.py:58` and the code agree after the change —
      whichever way the pin is expressed, there is exactly one version named in
      that file.
- [ ] **R4** No other behaviour change. Pace, caps, page selection and the
      returned markdown shape are untouched.

## Files

`backend/app/extraction/ocr.py` · `deploy/secrets.env.example` (only if R2
applies) · `backend/tests/` for the assertion below. Nothing else.

## Verify

- [ ] A test asserts the model string the OCR client is constructed with is the
      dated name, not an alias ending in `-latest`.
- [ ] `git grep -n 'mistral-ocr-latest' backend/` returns nothing outside a
      comment or a documented fallback.
- [ ] **State in the PR body whether `mistral-ocr-4-1` was confirmed live on
      the key** (`GET /v1/models`) at the time of the change, or that it was
      taken on the existing comment's word. Do not spend a call to find out if
      the CI environment has no key — say which it was.
- [ ] Target gates: `mypy app/` · `lint-imports` · `api-debt-check` ·
      `alembic heads` · `alembic-branch-check` · `test` — all green.

## Out of scope

Changing OCR vendor. Self-hosting. Requesting new response fields. Any
accuracy work — the audit's verdict is "yes-for-now" on the incumbent and this
ticket does not revisit it.
