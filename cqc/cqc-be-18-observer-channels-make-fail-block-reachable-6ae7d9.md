---
id: cqc-be-18-observer-channels-make-fail-block-reachable-6ae7d9
type: feat
status: queued
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 240, turns: 900, stallMinutes: 30}
depends: []
attempts: []
---
# Port the observer channels so a run can actually reach Fail-block

**The release-2 prerequisite. Without it the product cannot say "broken".** Sources: BE-9,
BE-28; `scripts/e2b/sandbox-agent-nav-go1.mjs` (the Go1 probe as it is today).

## The problem, in BE-9's words

> Release 1 and 2 verdicts are navigation-only. `main` has no SCORM observer and no
> JS-error / HTTP channel on the Go1 path, preview has `tracking="false"`, and
> `dependency_mode` is hard-coded `dynamic-external`. Almost every run is
> `Needs-review` / `limited`. Porting the observer channels is a named release-2
> prerequisite.

That is what the live dev data shows: of the three runs imported on 2026-09-29, one is
`Fail-block` and two are `Needs-review` — and the `Needs-review` verdicts are the
*absence of evidence*, not a judgement. Triggering more runs without this ticket just
produces more `Needs-review`.

## Scope

Port the three observer channels into the Go1 sandbox probe
(`scripts/e2b/sandbox-agent-nav-go1.mjs`), from wherever they already exist on the
non-Go1 / local path — find them first; do not write them from scratch if a working
implementation is already in the repo:

- **SCORM API observer** — calls made to the SCORM API surface by the package.
- **JS errors** — uncaught errors and unhandled rejections in the page.
- **HTTP ≥ 400** — failed requests the package makes, with URL and status.

Each channel's output must reach `playability-report.json` in a shape the existing report
consumer and the CMC's Evidence tab already understand. Read `src/common/` and the FE's
`PlayabilityReport` type before inventing a field.

## Acceptance criteria

- [ ] All three channels are captured on the Go1 path and appear in the published report.
- [ ] A run whose package errors reaches `Fail-block`; the existing `Needs-review` path is
      unchanged when the channels are silent.
- [ ] `REQUIRED_PLAYABILITY_CHECKS` and the `launch-renders` / `media-plays` check ids are
      reconciled if this changes which checks can conclude (BE-28) — or the PR says
      explicitly that it does not.
- [ ] Unit tests cover each channel's parsing, including the empty case.
- [ ] The redaction gate still passes: observer output can carry URLs and partner content.

## Blocked by

- (nothing)
