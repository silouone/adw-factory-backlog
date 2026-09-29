---
id: cqc-be-12-cors-preflight-on-every-cqc-route-7b41e2
type: bug
status: in-progress
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# Four of the five read routes reject the browser's preflight, so only the list is reachable

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints"; BE-15 (CORS is per stage).
Measured live on `dev` (`content-quality-checker-dev`) on 2026-09-29 with the CMC Content
quality page running on `http://localhost:3000`.

## The bug

`backend/serverless.yml` declares `cors:` on exactly one `http` event:

| function | route | `cors` |
| --- | --- | --- |
| `getContent` | `GET /cqc/content` | `{origins: …, headers: [Authorization, Content-Type]}` |
| `getConfig` | `GET /cqc/config` | **absent** |
| `getLoRuns` | `GET /cqc/content/{lo_id}/runs` | **absent** |
| `getCheck` | `GET /cqc/checks/{run_id}` | **absent** |
| `getArtefact` | `GET /cqc/checks/{run_id}/artefacts/{name}` | **absent** |

Without it, Serverless creates no `OPTIONS` method on those resources, so API Gateway
answers the preflight with `403` and the browser never sends the real request:

```
OPTIONS /cqc/content  ->  200, access-control-allow-origin: http://localhost:3000
OPTIONS /cqc/config   ->  403, no CORS headers
GET     /cqc/config   ->  200          (curl, Origin + staff bearer: the handler is fine)
```

The page lists content and nothing else: no check configuration, no run drawer, no run
history, no artefact. This is not a localhost problem — `staff-new.qa.go1.cloud` hits the
identical failure, so release 1 is unusable from any browser.

## Why the gates missed it

The *handlers* set `access-control-allow-origin` correctly and are unit-tested for it
(`decodeCorsOrigin`, `src/get-content/handler.test.ts`). But the **preflight is answered by
API Gateway, not by a lambda**, so no handler test can observe it. Only
`test/serverless-get-content.test.ts` asserts the serverless-level `cors` block; the other
four routes have no equivalent.

## Red first (Art. I)

Write the failing test before the fix: extend the `test/serverless-*.test.ts` family with a
**table-driven** assertion over every `/cqc` `http` event in the rendered `serverless.yml` —
each must declare `cors` with the stage's `corsAllowedOrigins` and the `Authorization`
and `Content-Type` headers. It must go red on four routes today.

## Acceptance criteria

- [ ] Every `/cqc` route declares the same `cors` block as `getContent`; the origins come
      from `custom.corsAllowedOrigins.<stage>` (BE-15), never a wildcard.
- [ ] One table-driven test enumerates the `http` events from the config itself, so a
      future route cannot be added without CORS and stay green.
- [ ] `config:check` passes and `serverless package --stage dev` still renders.
- [ ] No handler behaviour changes: the per-response CORS header logic stays as it is.

## Verify by hand after the deploy (operator)

`OPTIONS` each of the five routes with `Origin: http://localhost:3000`; all five return
`200` with `access-control-allow-origin` echoing the origin.

## Blocked by

- (nothing)
