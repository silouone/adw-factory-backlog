---
id: cqc-be-16-log-groups-expire-3c8d51
type: chore
status: in-progress
priority: 3
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 90, turns: 400, stallMinutes: 20}
depends: []
attempts: []
---
# The lambda log groups never expire

Found while inventorying the live `dev` stack on 2026-09-29
(adw-factory `ai_docs/2026-09-29-cqc-dev-aws-inventory.md`).

All eight `/aws/lambda/content-quality-checker--lambda-*-dev` log groups have
`retentionInDays: null` — never expire. Two reasons that matters:

- **Unbounded.** The refresher runs hourly forever. It is the only line in this stack's
  cost that grows without limit.
- **Retention.** The logs carry LO ids, run ids and Go1 user ids. "Keep forever" should be
  a decision, not an accident.

## Acceptance criteria

- [ ] Every lambda log group this service creates has an explicit retention. 30 days unless
      there is a reason to differ; say which you chose and why in the PR.
- [ ] Set it in `serverless.yml` (`provider.logRetentionInDays`) so it applies to every
      function, present and future — not eight per-function settings.
- [ ] A serverless-level test asserts the retention is set, so a future function cannot ship
      without one.
- [ ] `config:check` passes and `serverless package --stage dev` renders.

## Blocked by

- (nothing)
