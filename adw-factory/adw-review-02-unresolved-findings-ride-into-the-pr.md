---
id: adw-review-02-unresolved-findings-ride-into-the-pr
type: feat
status: done
priority: 1
review: false
created: 2026-09-18
depends: [adw-review-01-route-and-block-on-what-matters]
attempts: [{"runId":"adw-review-02-unresolved-findings-ride-into-the-pr-1789771187138","branch":"adw/adw-review-02-unresolved-findings-ride-into-the-pr","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-review-02-unresolved-findings-ride-into-the-pr-1789771187138/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/84","provider":"claude","model":"sonnet"}]
---
# Nothing a reviewer says may vanish

> **Spec authority:** `specs/adw-v1.12-review-routing.md` §4.
>
> Depends on `adw-review-01` because the routing/blocking split defines which
> findings are "unresolved but not fatal" in the first place.

## Why

`adw-v1.3` D1 already promised this:

> Judgement calls (including every baseline smell) and scope-creep notes do NOT
> route — **they ride into the PR body for the human.**

The first half shipped. The second half never did. A non-routed finding is
filtered out by `unionBlockingFindings` and never appears again outside
`runs/<runId>/journal.jsonl`, which is gitignored and lives only on the
operator's disk.

That is how `adw-fe-24`'s real defect stayed invisible: the reviewer found it,
said so three times, and nothing downstream of the filter ever mentioned it
again. The operator reviewing that PR had no way to know an objection existed.

After `adw-review-01`, this matters *more*, not less: `missing`, `unasked` and
`judgement` findings will no longer be able to block a run, so the PR body
becomes the only place a human can see them. Article IV's second end is the
PR — if the objection is not on the PR, the gate is not real.

## Requirements

- [ ] Every finding raised on either axis, of any kind or severity, that is
      **not resolved** by the end of the review loop appears in the PR body.
- [ ] Each rendered finding carries its `axis`, `kind` (spec only), `severity`,
      `file` (+ `line` when present), `citation` and `summary` — the same
      fields the typed verdict already guarantees (`adw-m9-01`).
- [ ] Blocking and advisory findings are visually distinguished. A reader must
      be able to tell "this was allowed to ship" from "this was fixed".
- [ ] Findings that were raised and then **fixed** during a fix round are not
      listed as unresolved. `adw-m9-05` already renders a `Fixed:` list — the
      two lists must agree and must not double-count.
- [ ] Zero unresolved findings renders **nothing** — no empty heading (the
      same N5 honesty rule `renderPreExisting` and `renderFindings` follow).
- [ ] The renderer is a pure function over the verdicts (Art. III), unit-tested
      without a live run, and wired into `prompts/pr-body.md` via `open-pr`.

## Verify

- [ ] Red test: a run ending with one advisory `missing` finding produces a PR
      body naming that finding, its file and its citation.
- [ ] Red test: a finding raised in round 1 and fixed in round 1 appears in the
      `Fixed:` list and **not** in the unresolved list.
- [ ] Red test: zero unresolved findings ⇒ no section, no stray heading.
- [ ] A finding's `citation` survives into the PR body verbatim — it is the
      quoted ticket line, and paraphrasing it defeats the point.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` — all green.

## Out of scope

- Changing which findings route or block — `adw-review-01`.
- The run screen's own rendering of the verdict (`adw-fe-21`, done).
