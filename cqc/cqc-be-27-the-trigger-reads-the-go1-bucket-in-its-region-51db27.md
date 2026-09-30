---
id: cqc-be-27-the-trigger-reads-the-go1-bucket-in-its-region-51db27
type: bug
status: in-progress
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# The trigger's preflight reads `go1-scormassets` from the wrong region, so every check will fail

Sources: BE-35, BE-36; `cqc-be-15` (same bug, refresher only). **Release 2 bug.** Found 2026-09-30.

## The bug

`backend/src/run-lifecycle/ports/index.ts:37` builds `new S3Client({})`. In Lambda that
resolves to `eu-west-1`. Lines 112-116 hand it to `createListAssetRevisions` for
`go1-scormassets`, which lives in **ap-southeast-2**. `common/ports/list-asset-revisions.ts:9-21`
sets no region. So once the Go1 grant lands, every `POST /cqc/checks` and `/cqc/batches`
preflight fails with `PermanentRedirect`, the error the refresher threw 39 times before `cqc-be-15`.

`cqc-be-15` fixed only `common/refresh-ports.ts:10-12` (`go1AssetRegion`). The constant is not
shared, so the lifecycle wiring never got the fix.

## Red first (Art. I)

Copy `common/refresh-ports.test.ts:54-66`. Assert that the S3 client in the production lifecycle
wiring (no injected client) resolves `config.region()` to `ap-southeast-2`. It is red today.

## Acceptance criteria

- [ ] One exported constant pairs the Go1 asset bucket with its region, and both the
      refresher and the lifecycle wiring import it. There is no second literal.
- [ ] The lifecycle's DynamoDB, SSM and Batch clients stay `eu-west-1`.
- [ ] A test fails if either wiring's asset client leaves `ap-southeast-2`.
- [ ] Fix the same stale fact in the docs:
      - `docs/backend/todo.md:18` and BE-36 (`decisions-2026-09-26.md:55`) list the grant roles
        as `{api,projector,refresher,runner}`. The roles that actually read the bucket are
        **runner, projector, refresher** (`s3:ListBucket` + `s3:GetObject`) and **lifecycle**
        (`s3:ListBucket` only, `serverless.yml:1255-1260`).
      - `api` never touches the bucket.

## Blocked by

- (nothing)
