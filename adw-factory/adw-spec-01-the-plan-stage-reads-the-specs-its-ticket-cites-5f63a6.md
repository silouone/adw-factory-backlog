---
id: adw-spec-01-the-plan-stage-reads-the-specs-its-ticket-cites-5f63a6
type: feat
status: queued
priority: 1
created: 2026-10-03
caps: {minutes: 150, turns: 300, stallMinutes: 25}
depends: []
attempts: []
---
# The plan stage reads the specs its ticket cites, and only the plan stage

> Operator decision 2026-10-03: specs live in the central ticket store at
> `~/adw/backlog/specs/<target>/`. Only the plan stage gets them. Build, test
> and repair inherit what `plan.md` distils.

## Context

- Specs cited by a ticket are only reachable when they happen to be committed
  in the target repo. adw-factory's in-repo `specs/` are read often (from
  session captures: `adw-v1.md` 62×, `v1.14` 49×, `v1.9` 46×), mostly in
  **build** sessions. Operator-local specs (for example
  `specs/adw-v1.17-mergeable-prs.md`, cited by 8 queued tickets) are invisible
  to every agent.
- Token cost is dominated by cache reads (98.3%, `ai_docs/2026-09-21-token-audit-HEADLINE.md`).
  A spec read early costs its size on **every later turn** of that session.
  A 5–10k-token spec in a ~70-turn build session costs about 0.5M tokens, while
  in a short plan session it costs far less. `.adw/artifacts/` is the only
  handoff between steps (adw-perf-05), so `plan.md` is the right carrier.
- The plan prompts (`feature-plan.md`, `bug-plan.md`) already render a
  `{{contextPointers}}` block. The chore lane has no plan stage.

## Requirements

- [ ] **R1 — store layout.** Specs live at `<backlog root>/specs/<target>/<name>.md`,
  where the backlog root is the parent of the target's ticket-store dir. A target
  with no central store (`ticketsDir` unset) gets none: a no-op, never an error.
- [ ] **R2 — cited specs only (pure).** A pure function extracts the spec
  references from the ticket body (`specs/<name>.md` tokens), deduplicated in
  order of first appearance. A reference that resolves to neither the central
  store nor the target repo's own `specs/` is listed as **missing** in the plan
  prompt, never silently dropped.
- [ ] **R3 — staging.** Before the plan stage's assemble, every cited spec found
  in the central store is copied into the workspace at
  `.adw/artifacts/specs/<name>.md`. Specs already in the target repo are pointed
  at in place, not copied. The staged files are not committed: the existing
  sweep pathspec already excludes `.adw/` (assert it).
- [ ] **R4 — plan prompt only.** The feat and bug plan prompts gain a "Specs this
  ticket cites" pointer block (paths only, never inlined content) plus the
  line: *"The spec is background; the ticket is the contract. Plan only this
  ticket, and carry into plan.md only the spec decisions this ticket needs."*
  No other stage's prompt names the specs.
- [ ] **R5 — journaled.** One event on the plan's assemble lists
  `{staged, inRepo, missing}`, so a run's spec exposure is answerable from the
  journal.
- [ ] **R6 — the backlog tab ignores `specs/`.** A test proves that the web
  backlog reader and every ticket-store scan (`status`, `next`, `clean`, sync)
  over a store whose root has `specs/<target>/*.md` list no spec as a ticket
  or as malformed.
- [ ] **R7** — once this lands, the 8 v1.17 tickets' spec lines (which
  currently say "operator-local, NOT in the repo") are updated to point at the
  central path. Operator or follow-up; record it in the PR.

## Out of scope

- Moving adw-factory's in-repo `specs/`. They stay versioned with the code.
- Giving agents read access to the backlog directory (it holds every target's
  tickets).
- Inlining spec content into any prompt.
- Measuring the quality effect. That's an adw-learn follow-up: compare review
  findings and repair rounds on spec-citing tickets before vs. after.

## Verify

- Pure tests for R2 (extraction, dedupe, missing). Lane-seam test (feat and
  bug): a ticket citing one central spec and one in-repo spec stages the first,
  points at the second, lists a third as missing, and the build/test prompts
  contain no spec path. A chore-lane test shows no staging.
- R6 store-scan tests.
- `bun run lint && bunx tsc --noEmit && bun run test` green.
