---
id: sabado-61-ingestion-commits-per-batch-9ed89f
type: feat
status: in-review
priority: 2
created: 2026-10-02
caps: {minutes: 300, turns: 1000, stallMinutes: 30}
depends: [sabado-32-the-extraction-stage-says-how-long-it-took-b1c44c]
attempts: [{"runId":"sabado-61-ingestion-commits-per-batch-9ed89f-1790978608205","branch":"adw/sabado-61-ingestion-commits-per-batch-9ed89f","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-61-ingestion-commits-per-batch-9ed89f-1790978608205/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"sabado-61-ingestion-commits-per-batch-9ed89f-1790996408477","branch":"adw/sabado-61-ingestion-commits-per-batch-9ed89f-4","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-61-ingestion-commits-per-batch-9ed89f-1790996408477/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"sabado-61-ingestion-commits-per-batch-9ed89f-1790988089832","branch":"adw/sabado-61-ingestion-commits-per-batch-9ed89f-3","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-61-ingestion-commits-per-batch-9ed89f-1790988089832/workspace","outcome":"in-review","provider":"claude","model":"claude-sonnet-5-5","pr":"https://github.com/App-sabado/sabado/pull/1362","rebaseRounds":1}]
---
# feat(extraction): a read commits as it goes, so a stop costs a batch instead of the whole mailbox

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3, "The
> 15-minute cap does not exist in code", and Wave E row 60. **This is the stated
> precondition for `ReadBudget`, and therefore for the 15-minute promise at all.**

## What happens today

**Nothing commits before `household_read.py:521`.** A stop anywhere earlier
loses everything the run did.

That is why the 15-minute cap **cannot be added yet**: it is not a gate, not a
deadline, not a budget, and it cannot become one while the only durable moment
is the very end. The only two clocks in ingestion are **absence-of-progress**,
so a live 3-hour run ticks progress and is never reaped. Past 15 minutes today,
the run simply continues.

Measured scale of what is at risk: Erwann's run is **2 h 27**, of which
extraction + founding + state + routing is **≈49%**.

## Requirements

- [ ] **R1** A read commits at batch boundaries, so a stop costs at most one
      batch. State the boundary you chose and why, in the PR body.
- [ ] **R2** A resumed or re-run read **does not redo committed work**, and does
      not double-write. `sabado-35` already established a seen-set discipline in
      the founding lane — reuse that idea rather than inventing a second one.
- [ ] **R3** Partial state is **legible**: a reader can tell a run that is still
      going from one that stopped half-done. A half-committed read that looks
      complete is worse than the current all-or-nothing.
- [ ] **R4** The commit boundary does **not** become a second progress clock.
      The two existing absence-of-progress clocks keep their meaning; this adds
      durability, not a bound.
- [ ] **R5** No change to what is extracted, to the prompts, or to the model
      calls. Pure durability.

## Files

`backend/app/extraction/household_read.py` and the lanes it drives ·
`backend/tests/`. No migration unless R3 requires one — if it does, say why in
the PR body.

## Verify

- [ ] A test interrupts a read mid-way and asserts the committed prefix survives
      and is marked as partial.
- [ ] A test asserts a re-run after an interruption does not duplicate rows.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

**The 15-minute cap itself, `ReadBudget`, `wave_plan` and `live_pending`** — all
Tier 1, all unstarted, all blocked on this. Any speed work. Changing the
absence-of-progress clocks.
