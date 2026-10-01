---
id: cqc-be-41-four-handlers-never-send-the-cors-header-6f2a84
type: bug
status: in-progress
priority: 1
created: 2026-10-01
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-be-41-four-handlers-never-send-the-cors-header-6f2a84-1790850925358","branch":"adw/cqc-be-41-four-handlers-never-send-the-cors-header-6f2a84","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-41-four-handlers-never-send-the-cors-header-6f2a84-1790850925358/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/53","provider":"codex","model":"gpt-6-sol"}]
---
# Four handlers return 200 without `access-control-allow-origin`, so the browser throws the response away

Measured against the deployed `dev` stack on 2026-10-01, with the CMC running on
`http://localhost:3000`. This is the **second half** of `cqc-be-12`: that ticket fixed the
`OPTIONS` preflight, which API Gateway serves. The header on the **actual response** comes
from the lambda, and four handlers never set it.

## The bug

```
GET … -H 'Origin: http://localhost:3000'            status   access-control-allow-origin
/cqc/content                                        200      http://localhost:3000
/cqc/config                                         200      MISSING
/cqc/content/{lo_id}/runs                           200      MISSING
/cqc/checks/{run_id}                                200      MISSING
```

A browser discards a cross-origin response with no `access-control-allow-origin`, **even on a
200**. The request succeeds, the data comes back, and the fetch still rejects.

Only `get-content` and `get-batch` apply it. `get-config`, `get-lo-runs`, `get-check` and
`get-artefact` do not:

```ts
// src/get-content/handler.ts:25-27  — the pattern the other four are missing
const corsOrigin = decodeCorsOrigin(event.headers, process.env.STAGE);
corsOrigin === null ? {} : { 'access-control-allow-origin': corsOrigin, vary: 'Origin' };
```

## What it breaks in the CMC, today

- **"Checks are unavailable because check configuration couldn't be loaded."** — `get-config`
- **"Couldn't load run details."** with every field `N/A` — `get-check`
- Run history in the drawer — `get-lo-runs`
- The whole Evidence tab — `get-artefact`

The list renders, which is why the page looks half-working rather than broken.

## Why the gates missed it, and why verification missed it too

No test asserts the header on a handler's success response; `get-content/handler.test.ts`
covers it for the one handler that has it. And the verification after `cqc-be-12` checked
`OPTIONS` on all six routes, saw `200` everywhere, and concluded CORS was fixed — the
preflight is only half of CORS, and the half it does not cover is the half that was broken.

## Red first (Art. I)

A table-driven test over **every** `/cqc` handler: given an `Origin` the stage allows, the
success response carries `access-control-allow-origin` echoing it and `vary: Origin`; given
an origin the stage does not allow, it carries neither. Error responses (401/403/404/500)
must do the same — a browser needs the header to read an error body too.

## Acceptance criteria

- [ ] All six handlers set the header on success **and** on every error path.
- [ ] `vary: Origin` accompanies it, so a cache cannot serve one origin's response to another.
- [ ] A disallowed origin still gets no header.
- [ ] The helper is applied in one place per handler, not copied six times — extract it if
      that is cleaner, but do not change `decodeCorsOrigin`'s behaviour.
- [ ] A test fails if a new `/cqc` handler ships without it.

## Verify by hand after the deploy (operator)

```
curl -s -D- -o /dev/null -H 'Origin: http://localhost:3000' -H "Authorization: Bearer $JWT" \
  "$API/cqc/config" | grep -i access-control-allow-origin
```
Then open the CMC drawer: run details, run history and the Evidence tab all load.

## Blocked by

- (nothing)
