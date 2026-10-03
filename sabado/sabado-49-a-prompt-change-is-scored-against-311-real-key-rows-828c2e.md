---
id: sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e
type: feat
status: blocked
priority: 2
created: 2026-10-02
caps: {minutes: 300, turns: 1000, stallMinutes: 30}
depends: []
attempts: [{"runId":"sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e-1791012552844","branch":"adw/sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e-1791012552844/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e-1791012552844","branch":"adw/sabado-49-a-prompt-change-is-scored-against-311-real-key-rows-828c2e","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1361","note":"hand salvage after E2BIG at assemble-test"}]
---
# feat(extraction): a prompt change is scored against 311 real key rows before it ships

> **Finding:** `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §3 and Wave
> C row 45.
>
> **`sabado-45` is a companion, not a blocker — considered and dropped as an
> edge.** The digest fixes *ledger attribution* (which stored reading came from
> which prompt); this scorer replays against a fixed corpus and produces its own
> run, so it does not need it. Land both; order is the operator's.

## What happens today

`sabado-prompt-lab` holds **311 golden key rows across four real mailboxes**, a
scorer that counts *invented* fields (`truth=NONE` counts against the model),
and an A/B replay whose control is `seeded_from: "shipped"` so it reads the live
prompt tree and **cannot drift**.

It is HEAD `9fcd422 wip`, working tree dirty, and **nothing in `sabado`
references it**. A prompt edit ships against no measurement at all.

> ⚠️ **Fix `replay.py:41-47` first.** It **silently drops nested container
> dicts**, which scored `situationMaritale` and `numeroFiscal` MISSING on
> **22/22 passes while the model read them correctly every time**. Vendoring the
> scorer without fixing this imports a known-false measurement and makes it
> authoritative.

## Requirements

- [ ] **R1** `keys/`, `corpus/` and `lab/{replay,score}.py` are vendored into
      this repo under `backend/app/extraction/eval/`, as **committed, reviewable
      files** — not a submodule, not a fetch at test time.
- [ ] **R2** **`replay.py:41-47`'s nested-container drop is fixed before
      anything is scored**, with a test naming `situationMaritale` and
      `numeroFiscal` as the cases that motivated it.
- [ ] **R3** The scorer's `truth=NONE` behaviour — an invented field counts
      against the model — is preserved exactly. It is the property that makes
      the instrument honest.
- [ ] **R4** It runs in CI as a gate on changes that touch the prompt tree.
      Scope it so it does not run on every PR unless it is fast — say in the PR
      body which it is and what it costs per run.
- [ ] **R5** The control stays `seeded_from: "shipped"`. **Do not freeze a
      prompt copy as the control** — the whole point is that it cannot drift.
- [ ] **R6** If a model call is needed to score, the gate must be **skippable
      without a key** and must say loudly that it skipped. A silently-skipped
      gate reads as green.

## Files

`backend/app/extraction/eval/` (vendored) · the CI workflow ·
`backend/tests/`.

## Verify

- [ ] A test asserts a nested container dict survives replay — the `replay.py`
      bug cannot regress.
- [ ] The scorer reproduces the prompt-lab's own baseline on at least one
      mailbox; **quote the figure and the mailbox in the PR body**, and say
      plainly if it does not reproduce.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Improving any prompt. The OCR golden set, which is `sabado-51`. Judging with a
model — `JUDGE-BRIEF.md` is explicitly *"a proposal, not an implementation"* and
stays one. `astrid` as a hold-out: it has **0 truths** and is not scoreable.
