---
id: cqc-be-00-redaction-scan-without-side-effects-4de841
type: chore
status: in-progress
priority: 1
created: 2026-09-28
review: false
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# The redaction scan can be imported without running the publisher

Prefactor for every writer into a CQC bucket (BE-17, `.agents/skills/cqc-guideline/` "One gate into a bucket").
Today the redaction scan (`SECRET_PATTERNS`, `scanText`, `redact`) lives in the context-pack
publisher, which runs `main()` at import time and shells out to `aws --profile`. The backend's
`import-run` (and later the release-2 runner) must reuse the same scan, so it moves first.
First item of `docs/backend/todo.md` "First build steps".

## What to build

Move the scan into its own side-effect-free module. The publisher imports it from there, so
there is still exactly one gate. Importing the new module must not read files, spawn
processes, or touch AWS.

## Acceptance criteria

- [ ] Importing the new module has no side effects (no `main()`, no child process, no AWS call).
- [ ] The publisher imports the scan from the new module; no duplicated pattern list remains.
- [ ] The publisher's `--selftest` stays green.
- [ ] A selftest (or test) for the new module covers: a clean text passes, each secret pattern is detected, and `redact` removes it.

## Verify

`node scripts/context-extract/publish-context-packs.mjs --selftest` and the new module's own selftest.

## Blocked by

None. This ticket can start immediately.
