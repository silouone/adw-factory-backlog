---
id: sabado-59-a-sealed-field-is-marked-so-the-fiche-stops-being-returned-whole-0098da
type: feat
status: queued
priority: 1
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-58-a-vault-document-is-present-but-sealed-not-omitted-8f1265]
attempts: []
---
# feat(chat): sealed-ness is a property of a field, not of one tool, so a fiche stops being handed over whole

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 5 (d) second
> bullet and (e) second bullet — marked `SEC:`. Re-verified on `main` at
> `c82ba2ec`.
>
> ## ⚠️ Operator decision required BEFORE this runs
> Two guards of this shape were **offered and declined on 2026-09-23**, on the
> stated position that a ZONE NORMALE field is findable by design. That decision
> is now shipped and live. **This ticket reverses it.** Do not dispatch until the
> operator has re-confirmed. If the 09-23 position stands, `reject` this ticket —
> it is a recorded decision, not an oversight.

## What happens today

`SEALED` is a hard-coded literal (`ai_tools_data.py:233`) applied by exactly one
tool — `get_bank_accounts`, which counts rows and never names the `*_enc`
columns. That part is genuinely good.

But sealed-ness is a property of **that tool**, not of a **field**. So:

```python
# backend/app/api/ai_tools_reads.py:423
#   extra_data holds the whole identity/health record. It is plaintext in ZONE NORMALE
# :433
"record": row.get("extra_data") or {},
```

`get_members` hands the model the fiche entire. Its own `limits` string in the
catalogue admits it: *« la fiche est rendue entière, y compris numéro de sécu et
allergies »*. In practice that means `numeroSecu`, `allergies`,
`medecinTraitant`, `numeroMutuelle` and `numeroFiscal` are readable by the chat
in the clear, today.

## Requirements

- [ ] **R1** A **sealed-field registry**, discovered at load time from one
      declaration — not a second hard-coded dict beside `ai_tools_data.py:233`,
      and not a list duplicated per tool. Read `backend/app/contracts/`
      (`field_aliases`, `contact_reference_keys`, `structural_keys`) first:
      **three working precedents for exactly this shape already exist. Do not
      invent a fourth.**
- [ ] **R2** A registered field comes back **marked, never omitted**, using the
      structural marker `sabado-42` landed (`core/sealed.py`) — the doctrine at
      `ai_tools_data.py:31-35`, applied where it currently is not.
- [ ] **R3** The registry covers `extra_data`'s identity and health keys at
      minimum: `numeroSecu`, `allergies`, `medecinTraitant`, `numeroMutuelle`,
      `numeroFiscal`. **List the full set you registered in the PR body** — the
      operator reviews the list, not the mechanism.
- [ ] **R4** Every tool that returns a raw record goes through the registry —
      `get_members` and `get_assets` both return `extra_data`-derived payloads
      (`ai_tools_reads.py:433`, `:555`). A new tool that forgets is a defect the
      registry should make hard, not a convention.
- [ ] **R5** The catalogue's `limits` strings stop claiming the fiche is returned
      whole, because it no longer is.

## Files

`backend/app/contracts/` (the registry, matching the existing three) ·
`backend/app/api/ai_tools_data.py` · `backend/app/api/ai_tools_reads.py` ·
`backend/app/api/ai_tools.py` (the `limits` strings) · `backend/tests/`.

## Verify

- [ ] A test asserts `get_members` never returns a registered field's value, and
      does return its marker.
- [ ] A test asserts the registry is read from one place — adding a key there
      changes every tool's output, with no second edit.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Moving any field into the coffre. Encrypting anything. Changing which fields the
**browser** shows — this is the model's read only. The document-level marking,
which is `sabado-58`.
