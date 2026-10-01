---
id: hq-09-ring-layout-maths-is-pure-and-tested-12319d
type: feat
status: in-review
priority: 3
created: 2026-10-01
depends: [hq-01-buildgraph-is-pure-and-tested-0aabb3]
attempts: [{"runId":"hq-09-ring-layout-maths-is-pure-and-tested-12319d-1790896775218","branch":"adw/hq-09-ring-layout-maths-is-pure-and-tested-12319d","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-09-ring-layout-maths-is-pure-and-tested-12319d-1790896775218/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/3","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The ring layout maths is pure and tested

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Extract the rings' geometry from the canvas code: sector spans from cluster weights (with the
top gap and inter-sector gaps), ring radii, the column-major placement of skills, the camera
clamp, and the zoom bounds k ∈ [1, 4.5]. The drawing is unchanged.

## Red first

- Sector spans plus the gaps sum to 2π; every sector is non-empty.
- Clamp: at k = 1, the pan is pinned to the origin; at k = 4.5, the rings' centre stays within the slack.
- The wheel zoom never leaves [1, 4.5], whatever deltas arrive.

## Acceptance criteria

- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-01-buildgraph-is-pure-and-tested-0aabb3
