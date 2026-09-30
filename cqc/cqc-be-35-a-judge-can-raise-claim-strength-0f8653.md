---
id: cqc-be-35-a-judge-can-raise-claim-strength-0f8653
type: feat
status: in-progress
priority: 3
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: []
---
# A judge's review maps to `claim_strength`, advisory until humans agree with it

Sources: FE D-25; schema review 1.3 (`docs/clxp-947-report-schema-review-2026-09-11.md:55-63`);
`docs/tech-review-2026-09-29-cqc-technical-state.md` §9.1, §9.4, §9.6.

**Build on PR #30 (`clxp-945-reviewer-agent`) once it is merged.** Reuse its binary-safe staging and scanning (`publish-context-packs.mjs`), `media-inventory.mjs` and `poc0/reviewer/*`. Do not re-implement them.

## Today

- `claimStrength()` is mechanical (`poc0/observability/playability-report.mjs:149-152`).
- `dynamic-external` caps it at `conditional`.
- `completedPath = false` (mapper :252).
- So every Go1 report is `limited`.
- The PR #30 judge outputs `TRUE | FALSE | UNCERTAIN` for six claims. It has **0 human
  labels**, and the LLM reviewers disagree with R0.

## Scope

- A pure `claimStrengthFromReview(review, mechanical) → strong | conditional | limited`,
  with table tests next to `playability-report.mjs`.
- The runner wiring after the report, behind a flag. **Default off.**
- While off, the result goes to an `advisory_claim_strength` sidecar. The report's
  `claim_strength` stays mechanical.
- The flag may only be turned on once human adjudication agrees on at least 9 of 10 packs.
  The PR must say this; it is not met today.

## Open decisions (operator; the code must make each a parameter, not a guess)

1. How the six claim outcomes map to the three levels.
2. Whether the judge may lift the `dynamic-external` cap, and a carve-out for the
   first-party Go1 CDN.
3. The model (the runner can invoke eu-central-1 inference profiles), and whether it fits the
   cost cap (about $0.90 per pack for four reviewers, tech-review §9.6, within the BE-24 caps).

## Red first (Art. I)

Table tests for `claimStrengthFromReview`, including "all UNCERTAIN → mechanical value".

## Acceptance criteria

- [ ] The pure function is covered by tests, and the mapping is data, not branches.
- [ ] With the flag off, the report is byte-identical to today's.
- [ ] Judge cost is recorded per run.

**Runner image:** this changes code that runs in the container. Add a `COPY` line in `scripts/runner/Dockerfile:28-34` for every new module (it copies files one by one). After merge the operator rebuilds and pushes the image, then deploys with `--param runnerImageTag=<commit>` (`scripts/runner/README.md:19-26`). Say so in the PR body.

## Blocked by

- PR #30 (`clxp-945-reviewer-agent`) merged (not a ticket)
