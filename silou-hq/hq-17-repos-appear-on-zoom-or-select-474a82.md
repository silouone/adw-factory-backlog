---
id: hq-17-repos-appear-on-zoom-or-select-474a82
type: feat
status: in-progress
priority: 3
created: 2026-10-02
depends: []
attempts: [{"runId":"hq-17-repos-appear-on-zoom-or-select-474a82-1790937434618","branch":"adw/hq-17-repos-appear-on-zoom-or-select-474a82","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-17-repos-appear-on-zoom-or-select-474a82-1790937434618/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/10","provider":"claude","model":"claude-sonnet-5-5"}]
---
# The fitted view shows no repo ticks; a cluster's repos appear when you zoom in or select it

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Story 10: "190 repos never clutter the fitted view". Today every repo is always placed on its cluster's arc and only shrunk ×0.7 at the fitted zoom (`src/web/entry.ts` ~576-581).

## Red first

- Pure `repoVisibility(k, selection, cluster)` (in the tested layout module, beside the ring maths): hidden at k=1 with nothing selected; shown for a cluster once k passes a threshold, or when that cluster or one of its repos is selected.
- When shown, the repos' angles stay within their cluster's arc (reusing the existing sector maths).

## Acceptance criteria

- [ ] The canvas draws repos only when `repoVisibility` says so. Badges and counts are unchanged.
- [ ] Reduced motion: repos appear with no fade or animation.
- [ ] Search and select of a repo still fly to it (the cluster expands first).
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- (nothing)
