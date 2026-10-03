---
id: sabado-45-an-edited-prompt-invalidates-the-ledger-it-wrote-305bf1
type: bug
status: in-review
priority: 1
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-45-an-edited-prompt-invalidates-the-ledger-it-wrote-305bf1-1791035444405","branch":"adw/sabado-45-an-edited-prompt-invalidates-the-ledger-it-wrote-305bf1","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-45-an-edited-prompt-invalidates-the-ledger-it-wrote-305bf1-1791035444405/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1383","provider":"claude","model":"claude-sonnet-5-5","rebased":"e88b13c792c07e67fb39657b15f14a8b6d57c154","ciRounds":1}]
---
# fix(extraction): an edited prompt mints a new extractor version, so the ledger stops mixing six readings as one

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3, "Two
> correctness bugs that make every accuracy number suspect", first bullet.

## What happens today

`extractor_version()` (`backend/app/extraction/enrich.py:2130`) carries **no
digest of the composed prompt bytes**. So **six shipped prompt versions ride
under one ledger version**: scalars v4 and lists v7–v11 each carry a
`stays_at_registry_version: registry-v8` note.

The v11 entry states the consequence itself: *"a mailbox already read holds no
SIRET until it is read again."*

> **The ledger is silently mixing readings produced by six different prompts.
> Any accuracy figure measured today is measured across two or more of them** —
> which makes every extraction number in the project provisional, including the
> ones the OCR strategy and the prompt-lab rest on.

This is the same failure class the chat side already hit twice (`RELEASE = 2`
dated after the run that minted it; `44/68` on an R6 tree). A version that does
not change when the thing it names changes is not a version.

## The red test

Compose the extraction prompt, record `extractor_version()`. Change one byte of
a prompt the composition reads. Assert `extractor_version()` differs. Today it
does not — that is the red.

## Requirements

- [ ] **R1** A digest of the **composed prompt bytes** enters
      `extractor_version()`. Read `backend/app/ai/release.py` and
      `chat_loop.common_fingerprint()` first — the chat side solved exactly this
      and its two-hash split is documented and reasoned. **Match that shape; do
      not invent a second versioning scheme.**
- [ ] **R2** The digest covers everything that reaches the model and nothing
      that does not — comments and docstrings must not mint a version, prompt
      content must. `chat_loop.tool_logic()` (a SHA-256 over stripped ASTs with
      docstrings removed) is the worked precedent.
- [ ] **R3** The version is **reproducible** — same bytes, same version, across
      processes and Python patch versions. `release.py:3-9` records why the chat
      side needed a second, reproducible half; honour the same constraint.
- [ ] **R4** Existing ledger rows are not rewritten. State in the PR body what a
      pre-digest row now looks like and how a reader tells it apart.
- [ ] **R5** No change to any prompt, to the registry, or to what is extracted.

## Files

`backend/app/extraction/enrich.py` · wherever the prompt tree is composed ·
`backend/tests/`.

## Verify

- [ ] The red test above is green.
- [ ] A test asserts a comment-only edit does **not** mint a version.
- [ ] A test asserts the same prompt bytes produce the same version in two
      separate processes.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Re-reading any mailbox. Backfilling the ledger. Changing the registry's own
numbering. Any accuracy work — this ticket makes accuracy *measurable*, it does
not improve it.
