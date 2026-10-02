---
id: sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33
type: bug
status: done
priority: 2
created: 2026-10-01
caps: {minutes: 90, turns: 300, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-1790808599352","branch":"adw/sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-1790808599352/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-1790842826374","branch":"adw/sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-1790842826374/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-1790898709761","branch":"adw/sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-3","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-42-sealed-is-a-structural-check-not-a-substring-test-b6ac33-1790898709761/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1333","provider":"claude","model":"claude-sonnet-5-5","ciRounds":1}]
---
# fix(chat): a turn is marked sealed by the marker it carries, not by the word appearing anywhere in it

> ## Base: PR #1283 is merged — this ticket is runnable
>
> `feat/chat-calls-tools` landed on `main` at **2026-10-01T06:01Z**, so `chat_loop.py`,
> `ai_tools*.py`, `usage.py` and `chat_turn.py` are all present. Line numbers below are as
> audited 2026-09-30 on the pre-merge branch and shifted in the rebase — **navigate by
> symbol, not by line**.
>
> **A first attempt was fired at 2026-09-30T22:49Z, seven hours BEFORE that merge**, cut
> from `29ae7af5` where none of these files existed. All four such runs blocked on
> `red-check exhausted` having produced nothing — an agent cannot write a failing test for
> a defect in a file that is not in its tree. The `attempts` entry below is that run. It is
> not a verdict on this ticket.

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 6, final note.
> Smallest ticket in the batch, and deliberately so: the observability audit
> (`2026-09-30-go1-observability-and-eval.md`, §The 59×-cost problem) calls it **"the
> poster child"** for why deterministic invariants outrank an LLM judge — *"an invariant
> catches it, a judge never would."* Fix the defect, and let the fix be the argument.

## What happens today

`chat_loop.py` ≈`:828`:

```python
"sealed": "sealed" in json.dumps(output)
```

The vault boundary's own doctrine (`ai_tools_data.py` ≈`:31-35`) states the law:
*"Omission is indistinguishable from absence."* The marker exists so a model can tell
« you have no lease » from « your lease is in the vault ». Whether a turn touched sealed
data is therefore a **fact about the marker**, and it is currently computed as a
**substring test over the serialised result**.

Two ways it is wrong, in both directions:

- **False positive.** Any document, note, contact name or calendar title containing the
  word *sealed* — in any language, in any field, at any depth — flips the flag. A person
  with a paper about a *sealed* bid marks every turn that reads it.
- **False negative.** If the marker's own key or shape is ever renamed, the substring may
  still match while the structure has moved, so the flag reads true for the wrong reason
  and nothing fails.

It is also the exact defect class an LLM judge cannot see: the answer to the person is
correct, so a judge scores the turn 5/5 and the flag stays wrong forever. A structural
assertion over the same data catches it in milliseconds, for free, on every turn.

## Requirements

- [ ] **R1** `sealed` is computed by inspecting the result **structure** for the sealed
      marker — the shape `ai_tools_data.py` defines (≈`:233`) — not by searching the
      serialised text.
- [ ] **R2** It walks nested structures. A sealed field inside a list of records counts;
      the marker is per-field, not per-result.
- [ ] **R3** The marker's shape is named in **one** place and both the producer
      (`ai_tools_data.py`) and this consumer read it from there. Today the producer has a
      hard-coded literal and the consumer has a string search — two independent
      definitions of the same fact.
- [ ] **R4** No change to what the person sees, to the tools' results, or to the vault
      boundary itself. This ticket changes a **derived flag**, nothing else.

## Files

`backend/app/ai/chat_loop.py` (the `sealed` computation) ·
`backend/app/api/ai_tools_data.py` (the single definition of the marker, R3) ·
`backend/tests/`. Nothing else.

## Verify

- [ ] **The red test, first.** A tool result containing the word `sealed` in a **data**
      field — a document's content, a contact's name — with **no** sealed marker present,
      asserts `sealed is False`. It must **fail on the current tree**; that failure is the
      ticket.
- [ ] A second red test: a genuinely sealed field **nested inside a list** asserts
      `sealed is True`. Verify whether the current implementation happens to pass this one
      and say so in the PR body — a substring test can be accidentally right.
- [ ] Regression: `get_bank_accounts`, the one tool that applies the marker today, still
      reports `sealed is True`.
- [ ] R3: `git grep` shows the marker's shape defined once.
- [ ] Target gates: `mypy app/` · `lint-imports` · `api-debt-check` · `alembic heads` ·
      `alembic-branch-check` · `test` — all green.

## Out of scope

The sealed-**field** registry discovered at load time, so `get_members`' `record` marks the
NIR and health fields it currently returns in the clear — a real `SEC:` gap and its own
ticket. Marking vault documents present-but-sealed in `_document_rows` instead of omitting
them — likewise. This ticket fixes the flag, not the boundary.
