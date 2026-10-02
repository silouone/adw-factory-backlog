---
id: hq-18-small-canvas-and-read-fixes-90c1f9
type: feat
status: in-progress
priority: 3
created: 2026-10-02
depends: [hq-17-repos-appear-on-zoom-or-select-474a82, hq-11-an-m1-scan-survives-a-missing-cache-933bf9]
attempts: [{"runId":"hq-18-small-canvas-and-read-fixes-90c1f9-1790968312541","branch":"adw/hq-18-small-canvas-and-read-fixes-90c1f9","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-18-small-canvas-and-read-fixes-90c1f9-1790968312541/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/14","provider":"claude","model":"claude-sonnet-5-5"}]
---
# Small canvas and read fixes: hover cluster, double-click fits, existing hooks, quoted ssh tail, factory name

> Spec: `~/personal_project/silou-hq/docs/spec-v1.md` (binding). Rules: `CLAUDE.md`. v1 is read-only: no write route, no action button.

The small gaps left from the v1 audit, grouped together. Each ships with a test where one is possible.

## Red first

- **Hook files exist** (story 15): a hook whose `~/.claude/hooks/<script>` file is absent is not allowlisted for `/file` (`hasFile: false`). A fixture with one present and one absent hook proves it (`build.ts` ~387).
- **Quoted ssh tail** (story 27): the M1 `/output` command passes its path as one shell-quoted argument. A pure command builder test with a path containing a space and a `'` (`server.ts` ~196).
- **Factory name** (story 34): the Factory widget heading renders `factory.name` from config, not the literal "adw-factory" (`widgets/factory.tsx` ~43).
- **Hover shows cluster** (story 7): the pure hover-fact function returns the cluster for skills and routines too (`inspect.ts` ~53-64).

## Acceptance criteria

- [ ] Double-click on the canvas always returns to the fitted view (story 6). The project page opens from the badge's inspector, not from a double-click (`entry.ts` ~805).
- [ ] Every Red-first item has a test.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Blocked by

- hq-17-repos-appear-on-zoom-or-select-474a82
- hq-11-an-m1-scan-survives-a-missing-cache-933bf9
