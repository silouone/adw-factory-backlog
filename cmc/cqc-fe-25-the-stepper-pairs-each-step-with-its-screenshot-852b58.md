---
id: cqc-fe-25-the-stepper-pairs-each-step-with-its-screenshot-852b58
type: feat
status: queued
priority: 2
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 180, turns: 700, stallMinutes: 25}
depends: [cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952]
attempts: []
---
# The snapshot stepper shows each step's screenshot next to its accessibility tree

Sources: FE D-8. Release 1 put this out of scope (`spec-cqc-fe-release-1.md:334-336`).
Backend: `cqc-be-33` publishes `snapshots/step-NNNN.png` numbered like the YAML.

The `cqc-be-*` blocker below lives in the CQC ticket store (`~/adw/backlog/cqc`), so it is **not** in `depends:`, because `just next` only resolves ids in the same store. Check that it is `done` (merged **and deployed to dev**) before dispatching.

## Red first (Art. I)

A `SnapshotStepper` test: a step with both `runs/snapshots/step-0003.yml` and
`runs/snapshots/step-0003.png` renders the image. A step with only the YAML renders as today.

## Acceptance criteria

- [ ] Pairing is by step number. A missing PNG is not an error.
- [ ] Images load lazily through the artefact presign endpoint, and only for the visible step.

## Blocked by

- cqc-fe-21-evidence-resolves-prefixed-artefact-names-84f952
- cqc-be-33-every-step-has-a-screenshot-5144c2 (CQC store)
