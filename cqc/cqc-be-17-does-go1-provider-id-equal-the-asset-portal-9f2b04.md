---
id: cqc-be-17-does-go1-provider-id-equal-the-asset-portal-9f2b04
type: chore
status: done
priority: 2
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-be-17-does-go1-provider-id-equal-the-asset-portal-9f2b04-1790691511926","branch":"adw/cqc-be-17-does-go1-provider-id-equal-the-asset-portal-9f2b04","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-be-17-does-go1-provider-id-equal-the-asset-portal-9f2b04-1790691511926/workspace","outcome":"in-review","pr":"https://github.com/CoorpAcademy/content-quality-checker/pull/29","provider":"codex","model":"gpt-6-sol","rebased":"9d70f6f0f83d02ce8538ebd6ff42e2ecba507b6f"}]
---
# Spike: is Go1's `core.provider_id` the same thing as the asset portal?

Release-2 prerequisite. Sources: BE-33, BE-11.

## Why it has to be answered before the trigger is built

BE-33: *"The asset portal comes from the run, not a Go1 API. The refresher only tracks LOs
that have runs, so `asset_portal_id` is always known. Whether Go1's `core.provider_id`
equals the asset portal only matters when triggering an LO that has never run."*

Release 2's whole point is triggering an LO that has **never run** — staff paste ids from
Slack. For those, there is no run to read `asset_portal_id` from, and the runner needs the
asset prefix `go1-scormassets/<env>/<asset_portal_id>/<lo_id>/` to find the package. If
`core.provider_id` is not the asset portal, the trigger cannot locate the package and
release 2 needs a different answer before any of it is built.

## What to produce

This is a **spike**: the deliverable is evidence and a recorded decision, not a feature.

- Take a sample of LOs that **do** have runs, so `asset_portal_id` is known from the run
  record. For each, fetch Go1's `core.provider_id` through the existing gateway client
  (`src/common/ports/`, the same path `cqc-be-08` uses) and compare.
- Report how many agree, how many differ, and what the differing ones look like. A single
  counter-example is the answer.
- If they agree everywhere in the sample, say how large the sample was and over what spread
  of portals — an agreement claim is only as strong as its spread.

## Acceptance criteria

- [ ] A written finding in `docs/backend/` with the sample size, the method, and the raw
      comparison table.
- [ ] A new decision `BE-39` recorded in `docs/backend/decisions-2026-09-26.md` stating
      whether `core.provider_id` can be used as the asset portal for a never-run LO, and if
      not, what release 2 must do instead.
- [ ] No production code changes. If the spike needs a throwaway script, put it under
      `scripts/` and say in the PR that it is throwaway.

## Blocked by

- (nothing — it reads the existing dev data and the Go1 gateway)
