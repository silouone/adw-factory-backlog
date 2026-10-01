---
id: cqc-be-46-errors-and-artefacts-reach-the-browser-e2fd2c
type: bug
status: done
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-be-46-errors-and-artefacts-reach-the-browser-e2fd2c-1790857514939","branch":"adw/cqc-be-46-errors-and-artefacts-reach-the-browser-e2fd2c","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-46-errors-and-artefacts-reach-the-browser-e2fd2c-1790857514939/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/54","provider":"codex","model":"gpt-6-sol"}]
---
# Gateway errors and evidence files never reach the browser: no CORS on 4xx/5xx, none on the artefacts bucket

Measured 2026-10-01 against the deployed `dev` stack, with the CMC on
`http://localhost:3000`, from a clean headless Chromium and from Node `fetch` with an
`Origin` header. Evidence: adw-factory `ai_docs/2026-10-01-cqc-fe-design-pass/`.

This is the **third and last** CORS gap. `cqc-be-12` fixed the `OPTIONS` preflight.
`cqc-be-41` (in progress) adds the header to the four handlers that omit it on the real
response (`get-config`, `get-lo-runs`, `get-check`, `get-artefact`). **Do not touch those
four handlers here.** Two gaps belong to nobody yet.

## Gap 1 — API Gateway's own error responses carry no CORS header

`backend/serverless.yml:706-724` defines `InvalidTokenGatewayResponse` (401) and
`NotStaffGatewayResponse` (403) with `ResponseTemplates` only — no `ResponseParameters`.
There is no `DEFAULT_4XX` or `DEFAULT_5XX` entry.

So every error the gateway answers itself — an expired token, a non-staff caller, a
throttle, an unknown route — is discarded by the browser as a CORS failure. The CMC
then shows "Check your connection and try again" for what is really "your session
expired", and cannot tell the user which.

```
curl -i -H 'Origin: http://localhost:3000' $DEV/cqc/config        # no Authorization
HTTP/2 401          (no access-control-allow-origin)
```

## Gap 2 — the artefacts bucket has no CORS configuration

```
aws s3api get-bucket-cors --bucket content-quality-checker--s3-artefacts-dev
→ NoSuchCORSConfiguration
```

`ArtefactsBucket` (`backend/serverless.yml:726-758`) has no `CorsConfiguration`. The CMC
evidence viewer gets a presigned URL from `GET /cqc/checks/{run_id}/artefacts/{name}` and
then `fetch`es it from the browser. That second request fails with `net::ERR_FAILED` for
**every** artefact (SCORM trace, page snapshot, accessibility snapshots), and the drawer
shows "Couldn't load this evidence. The link may have expired." — which is false; the link
is fine, the bucket refuses the origin.

## The fix

1. Gateway responses: add `ResponseParameters` setting
   `gatewayresponse.header.Access-Control-Allow-Origin` and
   `gatewayresponse.header.Vary: 'Origin'` on the two existing responses, and add
   `DEFAULT_4XX` and `DEFAULT_5XX` entries with the same headers. A gateway response cannot
   match the origin against a list; echo `method.request.header.Origin` (the CMC sends a
   bearer header, not cookies, so no credentials are involved).
2. Bucket: add a `CorsConfiguration` to `ArtefactsBucket` allowing `GET` and `HEAD` from
   `${self:custom.corsAllowedOrigins.<stage>}` — the same list the routes use
   (`serverless.yml:52-59`). No `PUT`, no wildcard origin.

## Acceptance criteria

- [ ] A red test first: extend the `serverless print` pattern of
      `backend/test/serverless-cors.test.ts` to assert, for every stage, that
      (a) the 401, 403, `DEFAULT_4XX` and `DEFAULT_5XX` gateway responses set
      `Access-Control-Allow-Origin`, and (b) `ArtefactsBucket` has a `CorsConfiguration`
      whose `AllowedOrigins` equals that stage's `corsAllowedOrigins` and whose
      `AllowedMethods` is exactly `GET`, `HEAD`. Watch it fail before the fix.
- [ ] No handler file changes (that is `cqc-be-41`).
- [ ] All gates green.

## Verify after deploy (operator)

```
curl -si -H 'Origin: http://localhost:3000' $DEV/cqc/config | grep -i access-control   # 401 now carries it
aws s3api get-bucket-cors --bucket content-quality-checker--s3-artefacts-dev           # rule present
```

Then open a run in the CMC drawer, Evidence tab, SCORM trace: the table renders.
