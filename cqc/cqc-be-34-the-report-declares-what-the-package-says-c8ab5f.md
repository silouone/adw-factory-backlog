---
id: cqc-be-34-the-report-declares-what-the-package-says-c8ab5f
type: feat
status: queued
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-be-39-the-go1-asset-grant-covers-every-stage-704a18]
attempts: []
---
# The report's declared half comes from the SCORM manifest, not an empty placeholder

Sources: FE D-24; BE-21/31, BE-32, BE-35/36; schema review 1.4. BE-39 does **not** bind this:
the run learns its asset portal from the launch claims (`sandbox-agent-nav-go1.mjs:163`) and
the revision from the frame URL (BE-32). BE-39 only covers the pre-trigger lookup.

**Build on PR #30 (`clxp-945-reviewer-agent`) once it is merged.** Reuse its binary-safe staging and scanning (`publish-context-packs.mjs`), `media-inventory.mjs` and `poc0/reviewer/*`. Do not re-implement them.

## Today

- The mapper hard-codes an empty `package_extraction_snapshot` and `content_hash: null`,
  and emits a limitation saying so (`playability-from-nav-run.mjs:258, 313-323`).
  A test asserts that text (:403).
- `poc0/manifest.mjs:29` (`parseManifest`) and :98 (`readDeclaredManifest`) already exist.
- The runner role has List and Get on the stage prefix (`backend/serverless.yml:906-925`).

## Change

- In the runner, `GetObject <env>/<configuration>/<lo>/<rev>/imsmanifest.xml`.
  - The client needs an **explicit `ap-southeast-2`**; the container defaults to `eu-west-1`
    (see `cqc-be-27`).
  - Reuse PR #30's bucket reader.
- Write `declared-manifest.json` and add it to `RAW_KEEP`.
- In the mapper, fill the snapshot and each check's `declared`, drop the limitation, and set
  `content_hash` from the listing's ETags.
- If the frame URL has no revision (1 in 12 sampled runs): keep today's behaviour and keep the
  limitation. Don't guess the revision.

## Red first (Art. I)

Mapper selftest: given `run.declared = parseManifest(fixture)`, the snapshot is non-empty
and the limitation is absent. Flip the assertion at :403.

## Acceptance criteria

- [ ] The declared half is filled when the manifest is found.
- [ ] If the manifest is missing, the run still succeeds, with the limitation.
- [ ] No second bucket reader alongside PR #30's.
- [ ] Whether "declared" media gates `media-*` stays an open decision: report the data,
      don't change verdicts here.

**Runner image:** this changes code that runs in the container. Add a `COPY` line in `scripts/runner/Dockerfile:28-34` for every new module (it copies files one by one). After merge the operator rebuilds and pushes the image, then deploys with `--param runnerImageTag=<commit>` (`scripts/runner/README.md:19-26`). Say so in the PR body.

## Blocked by

- PR #30 (`clxp-945-reviewer-agent`) merged (not a ticket)
- cqc-be-39-the-go1-asset-grant-covers-every-stage-704a18 (at least the dev grant)
