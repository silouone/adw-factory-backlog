---
id: cqc-be-19-the-runner-container-replaces-e2b-4d17bb
type: feat
status: queued
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: [cqc-be-18-observer-channels-make-fail-block-reachable-6ae7d9]
attempts: []
---
# The check runs in our own container, not an E2B sandbox

Sources: BE-19, BE-27; `scripts/e2b/build-template.mjs` (56 lines, the template definition),
`scripts/e2b/run-agent-nav-go1.mjs` (235 lines, the E2B SDK launcher),
`scripts/e2b/sandbox-agent-nav-go1.mjs` (573 lines, what runs inside).

## Why

BE-19: *"Triggered runs execute in a container on ECS Fargate submitted through AWS Batch;
no E2B in production. A Lambda cannot host even the E2B launcher (it lives up to 80 min),
so a container is required either way; E2B would only add a second runtime, shipped static
keys and a 1 h session cap."*

## Scope

- **Dockerfile**, derived from `scripts/e2b/build-template.mjs`: node:22 **x86**, Chrome,
  the pinned CLIs, aws CLI v2, `GIT_COMMIT` baked in at build time.
- **In-container launcher** replacing the E2B SDK calls in `run-agent-nav-go1.mjs`. The
  probe (`sandbox-agent-nav-go1.mjs`) should need as few changes as possible — it is the
  part that works.
- **Credentials from the chain**, not `aws configure get --profile`. BE-27: the publisher's
  `--profile coorp` becomes the default credential chain in the container; the runner role
  also needs Bedrock invoke in **eu-central-1**.
- **Publishing goes through the existing publisher** (`CQC_CONTEXT_DEST`), never a direct
  `PutObject`. The redaction gate is the only way into a CQC bucket (BE-17).

Out of scope: the Batch queue and compute environment (`cqc-be-20`), and the API that
submits jobs (`cqc-be-22`).

## Acceptance criteria

- [ ] `docker build` produces an image that runs one check end to end given `RUN_ID`,
      `LO_ID` and the stage, and publishes through the publisher.
- [ ] No E2B SDK import and no `E2B_API_KEY` on the production path.
- [ ] No `aws configure get --profile` anywhere in the container path.
- [ ] Exit codes are the contract `cqc-be-24` will read: **2 = preflight failed,
      3 = run failed**, 0 = published. Document them in the Dockerfile or a README next to it.
- [ ] `GIT_COMMIT` appears in the published `run-metadata.json` so a run can be traced to
      the image that produced it.
- [ ] The existing selftests stay green; the redaction gate still runs before publish.

## Blocked by

- cqc-be-18-observer-channels-make-fail-block-reachable-6ae7d9
