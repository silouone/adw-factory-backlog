---
id: sabado-43-the-chats-release-claim-states-what-it-ships-9e5a14
type: chore
status: in-progress
priority: 2
created: 2026-10-02
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: []
---
# chore(chat): the chat's quality claim names the release it actually ships

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md`, "Is 44/68
> defensible as merge-ready?" and Axis 1 (e).
>
> PR #1283 has since **merged** (2026-10-01). The stale claim merged with it and
> now lives in the tree.

## What happens today

The docstrings that shipped with the tool loop carry **R1 numbers on an R6+
tree**:

- `44/68` and the fingerprint `341cfc622ea9` appear in prose docstrings
  (`chat_loop.py:4`, `ai_tools_data.py:3-4`) and one arbitrary test constant
  (`test_chat_feedback.py:19`).
- The branch's own machinery disagrees with them: `release_label()` returned
  **`R6 · c010d4558879`** at merge, and `release_state()` computes `drifted`
  **precisely to catch this**.
- Five releases and ~60 commits sit between the claim and the code, including
  commits that changed tool *conduct*, the guard set, refusal semantics and the
  offered set.

And the claim is **unverifiable from this repo in principle**: the harness, the
corpus, the judge and the results ship in **no file here**. There is no `lab/`,
no scoring script, no results JSON — the instrument is a separate private repo
(`App-sabado/chat-lab-sabado`). `claude-opus-5`, "seven dimensions" and "zero
`missing-context`" have no in-repo referent whatsoever.

> A claim of provability that cannot be checked is **worse than no claim**. This
> is the exact drift `release.py` was written to prevent, and the bench already
> committed the same error once (`RELEASE = 2` dated **after** the last ledger
> turn).

## Requirements

- [ ] **R1** No docstring in `backend/app/` states a pass rate or a fingerprint
      that this repo cannot produce. Either strike the figure, or replace it with
      a **pointer** to where the measurement lives and the release it was taken
      at.
- [ ] **R2** Where a release is named, it is obtained from `release.py` /
      `release_label()` — **computed, never typed**. One version named per file.
- [ ] **R3** `test_chat_feedback.py:19`'s hard-coded fingerprint constant is
      either derived or clearly labelled an arbitrary fixture value, so nobody
      reads it as provenance again.
- [ ] **R4** A test asserts `release_state()` reports `drifted` when the tree
      moves past its minted release — the detector currently has **zero
      references** anywhere. This is the requirement that makes R1 stay true.
- [ ] **R5** No behaviour change. Prompts, tools, conduct and the fingerprint
      inputs are untouched.

## Files

`backend/app/ai/chat_loop.py` · `backend/app/api/ai_tools_data.py` ·
`backend/app/ai/release.py` (only if R2 needs a helper) ·
`backend/tests/test_chat_feedback.py` · `backend/tests/`.

## Verify

- [ ] `git grep -n '44/68\|341cfc622ea9' backend/` returns nothing, or only a
      clearly-labelled historical note.
- [ ] The R4 drift test exists and is green.
- [ ] **State in the PR body what `release_label()` returns on the merged HEAD.**
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Re-running the bench (the instrument is not in this repo). Vendoring the harness.
Changing any fingerprint input — that mints a release and is a different ticket.
