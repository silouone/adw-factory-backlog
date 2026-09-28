---
id: cqc-be-06-open-an-artefact-presigned-url-5ab9db
type: feat
status: queued
priority: 2
created: 2026-09-28
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: [cqc-be-05-lo-history-and-run-report-e7a7ca]
attempts: []
---
# Staff open a run's artefact through a short-lived link

Sources: `docs/backend/spec-cqc-backend-release-1.md` "Endpoints" (`GET /cqc/checks/{run_id}/artefacts/{name}`), "Hard rules".
Rules: `.agents/skills/cqc-guideline/` "Private evidence".

## Acceptance criteria

- [ ] Returns `{ url, expires_at, content_type }`: a fresh presigned GET valid 15 minutes.
- [ ] `name` must appear in the run's own listing; anything else (including `../`, absolute or encoded traversal) → `404 artefact_not_archived`. Tests cover the traversal cases.
- [ ] A cited artefact missing from the bucket → `404 artefact_not_archived`.
- [ ] The bucket stays private; nothing returns a public or long-lived URL.

## Blocked by

- cqc-be-05-lo-history-and-run-report-e7a7ca
