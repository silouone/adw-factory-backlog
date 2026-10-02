---
id: sabado-53-the-design-memos-survive-pr-236s-closure-cc3314
type: chore
status: queued
priority: 3
created: 2026-10-02
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: [sabado-47-an-extracted-fact-is-attested-against-the-bytes-the-model-saw-c4e601]
attempts: []
---
# chore(docs): the design memos survive PR #236's closure, because the code is replaceable and the measurement record is not

> **Finding:** `ai_docs/2026-09-30-sabado-gmail-poc-eval-harness.md`, surfaced in
> `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` Wave D row 52.
>
> **Depends on `sabado-47`** — the one piece of #236 worth keeping as *code* is
> the attester, and it should already be ported before the PR is closed.

## What happens today

PR #236 (`poc/gmail-wave1`) is **568 commits behind**, at alembic `0043` against
main's `0071+`, its backend payload has been duplicated by #374, and it carries
18 test scripts with no runner. It is not mergeable and should not be.

But it also carries the **July/August design memos and `DELIBERATION.md`** — the
record of why the ingestion design is shaped the way it is. Closing the PR
closes the only place that reasoning is written down.

> **The code is replaceable. The measurement record is not.**

## Requirements

- [ ] **R1** The July/August memos and `DELIBERATION.md` are committed to `main`
      as **code-free provenance** — documentation, under whatever docs path the
      repo already uses. No code, no scripts, no fixtures.
- [ ] **R2** They are committed **as written**, with a dated header saying where
      they came from and that they describe a prototype that was not merged.
      **Do not edit them into current truth** — a design memo whose reasoning has
      been retconned is worth nothing.
- [ ] **R3** Anything that reads as a current instruction is marked as
      historical, so no future agent follows a July plan as policy.
- [ ] **R4** Then **close PR #236**, referencing the commit that preserved its
      memos. State the PR number and the commit in the PR body.
- [ ] **R5** Nothing from #236 reaches `backend/app/` in this ticket.

## Files

The repo's docs path only. No `backend/`, no `frontend/`.

## Verify

- [ ] The memos are readable on `main` and each carries its provenance header.
- [ ] `git grep -l 'DELIBERATION' ` finds the preserved copy.
- [ ] PR #236 is closed with a comment pointing at it — **do the close as the
      last step, and say in the PR body that it was done**.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `frontend-check` · `test` — all green (a docs-only change should not move
      any of them).

## Out of scope

Porting any code from #236 — the attester is `sabado-47` and nothing else
qualifies. Reviving the branch. Rebasing it.
