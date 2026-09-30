---
id: cqc-be-45-fargate-navigation-can-pass-again-bfcb0d
type: bug
status: queued
priority: 1
created: 2026-09-30
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# A Fargate run can never pass `navigation-happened` since PR #30, and cites a HAR it never wrote

Sources: BE-9 (release-2 verdicts are navigation-only), PR #30 (`clxp-945-reviewer-agent`,
merged as `2eab94d`), its independent review of 2026-09-30. Release-2 regression: it reaches
production through the Batch runner image.

## The bug

PR #30 made `navigation-happened` stricter in the **shared** report builder
`poc0/observability/playability-from-nav-run.mjs:208-224`. It now passes only on
`observed.actionCorrelatedNavigation` (≥2 effective steps) **and** a complete
`observed.courseCoverage`. On main before the PR, the check passed on
`positions.transitions > 0`.

Only the E2B, local and `nav-evidence` paths produce those two fields. The Fargate container
entry, `scripts/e2b/sandbox-agent-nav-go1-container.mjs`, is a frozen copy of the pre-PR
launcher and produces neither. So on every Batch run the defaults at `:208-219` apply
(`available: false`), `navPass` is `false`, and the check is `Needs-review`. Under BE-9 that
was the one check a release-2 run could Pass, in preview and launch mode alike.

The same builder always declares `ev-har` → `network.har` (`:107`), and `media-reachable`
cites it (`:303`). The container never writes `network.har`, so every Fargate report cites
an artefact that doesn't exist. The reference check validates evidence ids, not files.

## The rule to implement (decided 2026-09-30)

PR #30's stricter rule is intentional: poller transitions alone are diagnostic, because a
timer or auto-advance can move the rendered position. **Do not weaken it where action
evidence exists.**

- **Action evidence was captured** (`observed.actionCorrelatedNavigation` present): keep PR
  #30's rule exactly.
- **Action evidence was never captured** (the field is absent, which is the container path
  today): fall back to main's release-2 rule, `positions.transitions > 0`. `observed.channel`
  must say `rendered-position-poller`, and a reason must say that action correlation was not
  captured on this runner, so a reader can tell the two bases apart.
- "Absent" means the field is missing. An E2B run that captured it with `available: false`
  is **not** eligible for the fallback.
- Declare `ev-har` only when the run actually captured a HAR. `media-reachable` must not cite
  a missing `ev-har`.

The real fix is porting Spike D's action correlation into the container. That is out of
scope here; open a follow-up ticket for it in your report.

## Red first (Art. I)

In the mapper selftest, feed a **container-shaped** run: `observed.positions` with
`transitions: 8`, no `actionCorrelatedNavigation`, no `courseCoverage`, no HAR. Expect
`navigation-happened` to Pass with channel `rendered-position-poller`, and no `ev-har` in
the evidence or in any `evidence_refs`. This goes red on main today.

Keep, or add, the opposite case: an E2B-shaped run with
`actionCorrelatedNavigation: {available: false, …}` and `transitions: 8` must stay
`Needs-review`. That guards against the fallback leaking.

## Acceptance criteria

- [ ] A container-shaped run passes navigation on `transitions > 0`, and its channel says so.
- [ ] An E2B or local run that captured action evidence is judged by PR #30's rule, unchanged.
- [ ] No report cites `network.har` unless the run wrote it.
- [ ] Nothing changes in the container launcher or the runner image; this is a mapper-only
      fix. It still needs an image rebuild, push and deploy with `runnerImageTag`, because
      the mapper ships in the image. Say so in the PR body.
- [ ] All target gates pass.

## Blocked by

- (nothing)
