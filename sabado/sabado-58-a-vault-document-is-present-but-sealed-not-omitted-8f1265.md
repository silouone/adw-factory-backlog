---
id: sabado-58-a-vault-document-is-present-but-sealed-not-omitted-8f1265
type: bug
status: queued
priority: 1
created: 2026-10-02
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# fix(chat): a sealed document comes back marked, not missing, so the model stops confusing « au coffre » with « tu n'en as pas »

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 5 (d), third
> bullet — marked `SEC:`. Re-verified on `main` at `c82ba2ec`.

## What happens today

`backend/app/api/ai_tools_data.py` states the rule as law at `:31-35`:

> **The sealed marker.** A tool never omits what it may not read. Omission is
> indistinguishable from absence. … A sealed value comes back as
> `{"sealed": true, "why": …}` and the model has nothing to guess.

And then breaks it, 570 lines later, in `_document_rows`:

```python
# :604
#   A SEALED row is not in this list, by the library's own rule — the coffre has its own …
```

So `get_documents` **omits** vault documents. The model cannot tell *"you have no
lease"* from *"your lease is in the coffre"*, which is the exact failure the
doctrine at `:31-35` was written against. The exclusion is **right for a browser
UI** — the coffre has its own two-tab listing — and **wrong for a model**, which
has no second tab to look in.

This is measured, not theoretical: `refuse-vault` is the second-worst case family
in the bench — **15 cases, 3 ok (20%)**, with 4 of the failures labelled
`missing-context` precisely because an absent row read as an absent document
(`ai_docs/2026-09-30-sabado-chat-state.md` §4.2, §7).

## The red test

`get_documents` is called for a household owning exactly one sealed document and
no others. Assert the returned rows contain one entry carrying the sealed marker.
Today the list is empty — that is the red.

## Requirements

- [ ] **R1** A sealed document appears in the model's document list, marked with
      the structural marker `sabado-42` landed (`backend/app/core/sealed.py`) —
      not a string, not a substring, not a bare boolean invented here.
- [ ] **R2** The marked row carries **enough to name the thing and nothing that
      reveals it**: whatever minimum identifies it to a person (its kind, its
      date) and a `why`. No filename if the filename is itself the secret — state
      the call made and why, in the PR body.
- [ ] **R3** The **browser's** two-tab listing is untouched.
      `api/shared_documents.py:416-418`'s exclusion is the library's own rule and
      stays exactly as it is. This ticket changes the model's read, only.
- [ ] **R4** The prompt is told what a marked row means, in the section
      `sabado-37` added — one sentence, in the existing voice, not a new section.
- [ ] **R5** No zero-knowledge property moves. The chat still imports no `Vault*`
      model; the test at `chat_context.py:3-5` still passes unchanged.

## Files

`backend/app/api/ai_tools_data.py` · `backend/app/api/ai_tools_reads.py` ·
`backend/app/ai/prompts_chat/chat-tools.fr.md` · `backend/tests/`.

## Verify

- [ ] The red test above is green.
- [ ] A test asserts no `Vault*` model is imported by the chat's read path.
- [ ] A test asserts the encrypted payload never appears in a tool result.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Marking sealed **fields** inside `get_members`' record — that is
`sabado-59-a-sealed-field-is-marked-so-the-fiche-stops-being-returned-whole-0098da`.
Any change to how sealing is set or reversed. The coffre's own screens.
