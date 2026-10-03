---
id: adw-web-03-pr-chips-show-conflicts-behind-and-parked-f5cd54
type: feat
status: in-progress
priority: 2
created: 2026-10-03
depends: []
attempts: [{"runId":"adw-web-03-pr-chips-show-conflicts-behind-and-parked-f5cd54-1791033594523","branch":"adw/adw-web-03-pr-chips-show-conflicts-behind-and-parked-f5cd54","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-web-03-pr-chips-show-conflicts-behind-and-parked-f5cd54-1791033594523/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/195","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The board's PR chips show conflicts, behind and parked before the operator clicks

> Spec: `specs/adw-v1.17-mergeable-prs.md` (operator-local, NOT in the repo — this ticket is self-contained; do not look for it), D9, stories 40–43.

## Context

The PR index (60 s TTL) queries
`number,state,headRefName,statusCheckRollup,reviewDecision`. It has no merge
state, so a conflicting PR such as #174 renders exactly like a shippable one.

## Requirements

- [ ] **R1** — the PR index's `gh pr list` query adds
  `mergeStateStatus,mergeable,isDraft,labels`. The parser validates them like
  its existing fields (one malformed row throws, with the target named).
- [ ] **R2 — pure projection.** It produces `conflicts` for `DIRTY` or
  `mergeable == CONFLICTING`; `behind` for `BEHIND`; `parked` for a draft
  carrying the `adw:needs-rebase` label (use the adw-merge-03 constant if it
  has landed, otherwise define it here and let adw-merge-03 import it); and
  **no chip** for `UNKNOWN` or anything else. `parked` outranks `conflicts`.
- [ ] **R3** — the card renders the chip next to the existing validation chip,
  in the design system's existing chip style. No new colours.
- [ ] **R4** — the web stays read-only (the negative-capability guard still
  passes).

## Out of scope

- Any action button (rebase now and the like).

## Verify

- Pure projection tests over every combination; the parser test with the new
  fields; a render test for one card per chip.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
