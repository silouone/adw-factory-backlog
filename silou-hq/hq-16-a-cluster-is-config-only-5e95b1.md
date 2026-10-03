---
id: hq-16-a-cluster-is-config-only-5e95b1
type: feat
status: in-review
priority: 3
created: 2026-10-02
depends: [hq-13-m1-routines-report-their-failures-too-eb305c]
attempts: [{"runId":"hq-16-a-cluster-is-config-only-5e95b1-1790989631743","branch":"adw/hq-16-a-cluster-is-config-only-5e95b1","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-16-a-cluster-is-config-only-5e95b1-1790989631743/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/18","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Adding a cluster in `hq.config.json` alone gives it routines, a colour and its repos

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

Story 47 is only half met. These are still hard-coded:
- the routine → cluster regex chain in `src/graph/build.ts` (`routineCluster`);
- the cluster colours in `src/web/hue.ts`;
- the repo-key prefixes in `build.ts`;
- the M1 host the scanner prints (`"host":"m1max"`).

## Red first

- A fixture config with a new cluster `{id:"zeta", match:"zeta"}` and a routine labelled `com.silou.zeta.daily` gives that routine `cluster: "zeta"`, through the same first-match regex rule as repos and memory.
- A cluster with no configured colour gets a deterministic one (same id → same hue across builds), distinct from its neighbours.
- The snapshot's host comes from `config.m1.host`, not a literal.

## Acceptance criteria

- [ ] No cluster id appears as a literal in `src/` outside defaults and tests (grep-tested).
- [ ] An optional `hue` per cluster in config overrides the derived one.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-13-m1-routines-report-their-failures-too-eb305c
