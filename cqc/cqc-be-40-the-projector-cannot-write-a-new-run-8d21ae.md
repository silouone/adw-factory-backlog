---
id: cqc-be-40-the-projector-cannot-write-a-new-run-8d21ae
type: bug
status: in-review
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-be-40-the-projector-cannot-write-a-new-run-8d21ae-1790850211906","branch":"adw/cqc-be-40-the-projector-cannot-write-a-new-run-8d21ae","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-40-the-projector-cannot-write-a-new-run-8d21ae-1790850211906/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/52","provider":"codex","model":"gpt-6-sol"}]
---
# The projector cannot write any new run: its role lost `dynamodb:PutItem`

Measured on the deployed `dev` stack, 2026-10-01, by importing 21 legacy runs. **Every one
failed.** The objects reached S3; not a single row reached the read model.

## The bug

`src/common/ports/transact-run-transition.ts:45` writes through `TransactWriteCommand`, and
its first `TransactItems` entry is a **`Put`** of the run item. AWS requires the underlying
action permission for each operation inside a transaction: a `Put` inside
`TransactWriteItems` needs `dynamodb:PutItem`, not just `dynamodb:TransactWriteItems`.

The projector role has the transaction permission and not the Put:

```
content-quality-checker--role-projector-dev
  dynamodb:GetItem  dynamodb:Query  dynamodb:TransactWriteItems  dynamodb:UpdateItem
                                     ^^^ no dynamodb:PutItem
```

Result, 32 consecutive invocations:

```
RunTransitionWriteError: publish failed for RUN#local-nav-11192471-... LO#11192471:
AccessDeniedException
```

`content-quality-checker--role-lifecycle-dev` does the same transaction and **does** have
`dynamodb:PutItem`, so this is a gap in one role, not a design question. The projector's
policy was not updated when it moved to the transition writer.

## Why nothing looked broken

The three runs already in the dev read model were imported **before** the transition writer
shipped. They still read fine, so every read endpoint, the CMC page and all verification done
so far looked healthy. The failure only appears when a *new* run is projected — which is also
the path the first real Fargate check will take.

## Why the gates missed it

Unit tests mock the DynamoDB client, so the IAM policy is never exercised. Nothing in the
suite compares the actions a port actually calls against the actions its role is granted.

## Red first (Art. I)

Assert it where a test can see it: for each lambda that writes through `TransactWriteCommand`,
the rendered `serverless.yml` role must grant the action behind **every** operation the
transaction contains — `Put` → `dynamodb:PutItem`, `Update` → `dynamodb:UpdateItem`,
`Delete` → `dynamodb:DeleteItem`, `ConditionCheck` → `dynamodb:ConditionCheckItem`. A
table-driven test over the roles is better than one assertion on the projector, because the
same gap can reappear on any new writer.

## Acceptance criteria

- [ ] The projector role grants `dynamodb:PutItem` on the runs table.
- [ ] Every other role that uses `TransactWriteCommand` is checked and fixed the same way;
      name each in the PR, including the ones already correct.
- [ ] A serverless-level test fails if a transaction-writing role is missing the action for
      an operation its transaction performs.
- [ ] No role gains a wildcard action or a wildcard resource.
- [ ] `config:check` passes and `serverless package --stage dev` renders.

## Verify by hand after the deploy (operator)

Re-import one legacy run and confirm a `RUN#` and `LO#` row appear:

```
npm run import-run -- --stage dev 11192471/local-nav-11192471-20260923-175123-8e936d
```

The 21 runs imported on 2026-10-01 are already in the dev bucket; re-importing is the
cheapest reproduction.

## Blocked by

- (nothing)
