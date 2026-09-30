---
id: cqc-be-28-config-says-which-case-actions-exist-40185e
type: feat
status: in-progress
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: []
---
# `/cqc/config` tells the CMC which case actions exist, so no button calls a missing route

Sources: BE-7; CMC spec "Unavailable actions are disabled and explain why"
(`spec-cqc-fe-release-1.md:381-387`); the `cqc-fe-16` decision (a), taken by the operator on 2026-09-30.

## Why

Resolve, Validate and Reopen render live in the CMC and call `POST`/`DELETE
/cqc/content/{lo}/resolution`. That route does not exist yet (`cqc-be-30`), so each click gets
a raw API Gateway `403`.

## Shape

Add `actions` to the config response:

```
actions: {
  resolve: { enabled: boolean, disabled_reason?: string },
  reopen:  { enabled: boolean, disabled_reason?: string },
}
```

Resolve covers Validate too. **Compute it in code, not SSM.** The flag must describe the
handlers that are actually deployed; an SSM value could claim a route that isn't there. Until
`cqc-be-30` ships, both are `enabled: false` with the reason `Resolution arrives in release 3`.

## Facts

- The config is decoded by a strict literal codec (`backend/src/common/decoders.ts:126-177`).
- `exactType` strips unknown keys (`decoders.ts:39`; test at `decoders.test.ts:164`).
- OpenAPI: `backend/openapi.yaml:538-597`.
- The handler is `backend/src/get-config/`.

## Red first (Art. I)

A `get-config` handler test: the response carries `actions.resolve` and `actions.reopen`,
each `enabled: false` with a non-empty `disabled_reason`. It is red today.

## Acceptance criteria

- [ ] `GET /cqc/config` returns `actions` exactly as above. `openapi.yaml` documents it, and
      `openapi:check` passes.
- [ ] The values come from code next to the route table. A test fails if a route is added
      without flipping its flag.
- [ ] The existing `launch_modes` and `verdict_scope` are unchanged.

## Blocked by

- (nothing)
