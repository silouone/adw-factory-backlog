---
id: cqc-be-15-the-go1-asset-bucket-is-in-sydney-a71d3c
type: bug
status: queued
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# The Go1 asset bucket is in ap-southeast-2, so every refresher run fails on the region

Sources: BE-35 (`dev` and `staging` read `go1-scormassets/qa/`), `docs/backend/todo.md`
"Owned by Silou" (the DevOps grant). Measured on the live `dev` stack, 2026-09-29.

## The bug

`src/common/refresh-ports.ts:10` hard-codes the Go1 asset client's region:

```ts
const defaultAssetClient = new S3Client({ region: 'us-east-1' });
```

The bucket is not in `us-east-1`. S3 says so itself:

```
$ curl -sI https://go1-scormassets.s3.amazonaws.com/
HTTP/1.1 403 Forbidden
x-amz-bucket-region: ap-southeast-2
```

So the hourly refresher has failed on every invocation since the stack went up — 39 errors
in the first four hours:

```
AssetRevisionListError: ListObjectsV2 revisions failed for
s3://go1-scormassets/qa/9494674/10622213/ : PermanentRedirect
```

`PermanentRedirect` is S3's "wrong region", not "no access". This is a **second, independent**
blocker behind the DevOps grant: even once Go1 grants read access, the call fails.

The same wrong region is recorded in `docs/backend/todo.md` ("Go1 account `769005983919`,
us-east-1"). That line is what the grant request was written from, so it needs the same
correction — flag it in the PR body; the operator owns the conversation with Go1 DevOps.

## Why the gates missed it

Every unit test constructs its own `S3Client` and asserts on `aws-sdk-client-mock`, so the
region in `refresh-ports.ts` is never exercised. Nothing in the suite asserts which region
the asset client is configured for.

## Red first (Art. I)

Assert the configured region of the asset client the production wiring builds — not one a
test constructs. `createRefreshPorts()` with no injected client must produce a client whose
resolved region is `ap-southeast-2`. It must go red today.

## Acceptance criteria

- [ ] The Go1 asset client resolves to `ap-southeast-2`; the DynamoDB client stays
      `eu-west-1` and the CQC artefacts bucket stays `eu-west-1`.
- [ ] The region is not duplicated in a second place — one named constant next to the
      `go1-scormassets` bucket name, so the bucket and its region cannot drift apart.
- [ ] A test fails if that constant changes, and its name says why the value is what it is.
- [ ] `docs/backend/todo.md`'s DevOps-grant line is corrected to `ap-southeast-2` in the
      same PR, and the PR body says the grant request may have been raised with the wrong
      region.
- [ ] `prod` is considered: BE-35 has prod read `go1-scormassets/production/` from the same
      bucket, so prod gets the same region. Do not introduce a per-stage region.

## Verify by hand after the deploy (operator)

The refresher's next hourly run no longer raises `PermanentRedirect`. It will still fail —
with an access error — until the DevOps grant lands; that is expected and is verification
item 6, not this ticket.

## Blocked by

- (nothing)
