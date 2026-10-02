---
id: sabado-56-a-tool-refuses-another-households-row-proven-per-family-cff226
type: feat
status: blocked
priority: 2
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-56-a-tool-refuses-another-households-row-proven-per-family-cff226-1790977929218","branch":"adw/sabado-56-a-tool-refuses-another-households-row-proven-per-family-cff226","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-56-a-tool-refuses-another-households-row-proven-per-family-cff226-1790977929218/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# test(chat): a tool handed another household's row refuses it, proven once per family

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md`, "Test coverage of
> the new loop", last bullet. Re-verified on `main` at `c82ba2ec`.

## What happens today

Per-account scoping in the tool layer is **sound by design**: every read is
either the router's own function or a scoped `SELECT` with
`user_id.in_(readable_owner_ids(db, user, category))`, and the documented rule is
*"never `db.get(Model, id)` on its own, which would read any account's row"*
(`ai_tools_data.py:323-326`). Every write calls the router function, which
applies `can_write`.

**None of that is tested negatively.** The three isolation tests #1283 added are
all route-level (`/chat-feedback/{id}`, `/usage/tokens`). There is **no case
handing a tool a foreign `event_id`, `asset_id` or `doc_hash`**. The property
the whole permission model rests on is held by convention and by review.

## Requirements

- [ ] **R1** One negative test per **tool family** — agenda, members, assets,
      contacts, documents, budget/expenses, child activities, bank accounts.
      Two households, and a tool handed the other one's identifier.
- [ ] **R2** The refusal is asserted as a **refusal**, not as an empty result
      where the distinction matters — and where the product's answer *is*
      "absence", the test says so explicitly and cites the rule it is honouring.
- [ ] **R3** Writes are covered as well as reads. `update_asset`,
      `update_member`, `update_contact` on a foreign row must not write.
- [ ] **R4** A **shared-grant** case: where a category is shareable, a grant
      makes the read succeed, and its absence makes it fail. `contacts` uses
      `user_id ==` because it is not shareable (`ai_tools_data.py:405-412`) —
      assert that too, so the asymmetry is pinned rather than remembered.
- [ ] **R5** The tests are structured so a **new tool added without scoping
      fails them** — a table-driven sweep over the catalogue, not eight
      hand-copied cases that a ninth tool silently escapes.

## Files

`backend/tests/` only. **No change to `backend/app/`** — this ticket pins
existing behaviour. If a real isolation hole is found, **stop and say so in the
PR body**; do not fix it here.

## Verify

- [ ] R5's sweep enumerates the catalogue, so `grep -c '_tool("'` and the number
      of cases agree.
- [ ] Each test fails if its scoping clause is removed — demonstrate for at
      least two families and say so in the PR body.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Changing any scoping rule. The sharing model itself. Route-level isolation,
which already has three tests.
