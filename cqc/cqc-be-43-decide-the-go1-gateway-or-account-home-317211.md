---
id: cqc-be-43-decide-the-go1-gateway-or-account-home-317211
type: manual
status: queued
priority: 3
created: 2026-09-30
depends: []
attempts: []
---
# Decide where CQC lives long term: behind a Go1 gateway prefix, or in a Go1 AWS account

**Decision ticket (operator, with Go1 platform and DevOps).** Sources: BE-2
(`decisions-2026-09-26.md:28`: "a later migration, redeploy + bucket policy, not a rewrite"),
CLXP-943 (still "DECISION NEEDED", `docs/tech-review-2026-09-29-cqc-technical-state.md:83`).

## To decide

- (a) A Go1 API gateway prefix in front of the Coorp stack, or
- (b) a redeploy into a Go1 AWS account.

Either choice touches:

- Hardcoded regions: `common/refresh-ports.ts:11`, `run-lifecycle/ports/index.ts:39`,
  `scripts/runner/run-agent-nav-go1.mjs:178`.
- Role names (`serverless.yml:17-32`) and the Go1 bucket grant (`cqc-be-39`).
- DNS, CORS, and the CMC's `APP_CQC_API_URL`.

## Done when

- [ ] The decision is recorded as a BE decision, with an owner and a date.
- [ ] Execution tickets are written from it.

## Blocked by

- (nothing)
