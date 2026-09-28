---
id: cqc-fe-02-gated-content-quality-page
type: feat
status: in-review
priority: 1
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-02-gated-content-quality-page-1790563990079","branch":"adw/cqc-fe-02-gated-content-quality-page","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-02-gated-content-quality-page-1790563990079/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"},{"runId":"manual-salvage-2026-09-28","branch":"adw/cqc-fe-02-gated-content-quality-page","outcome":"in-review","provider":"manual","pr":"https://github.com/go1com/domain-content-content-management-console/pull/136","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-02-gated-content-quality-page-1790563990079/workspace"}]
---
# An internal Content quality page is reachable from the CMC menu

First slice of the Content quality page (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 1 and 4, "Page shape" section). It establishes the route, the menu
item and the runtime configuration every later ticket builds on.

**Amended 2026-09-28 (operator decision): no Statsig gate.** Go1 ENG confirmed that
the CMC is an internal tool and ships new features without a Statsig feature gate
(Statsig is used on customer-facing apps like Go1 Learn). The Content quality page
ships as an "internal v1", visible to every CMC user. No Statsig dependency, hook,
client key or gate id; no not-found behaviour for a closed gate.

## What to build

- **Runtime configuration** for one new key, the CQC API base URL. It follows
  the existing path: container env → nginx SSI → `window.GO1` → the app config
  module. An unset or unreplaced value (`(none)`, blank, a literal
  `%REACT_APP_…%`) reads as not configured. Add it to `.env.example` with a
  comment and no real value.
- A **"Content quality" left-menu item** and a route in the flat route table.
  The path must not start with `/content`, because the menu marks items active by
  path prefix and the existing "Content" item would light up too.
- The route renders a single root implementation from a new feature folder
  under `src/components/`. For now it shows the page header and, when the CQC
  base URL is not configured, a clear "Content Quality Checker API is not
  configured" message instead of content.

## Acceptance criteria

- [ ] The menu shows a "Content quality" item linking to the page.
- [ ] The route renders the Content quality page header.
- [ ] CQC base URL missing, blank, `(none)` or unreplaced: the page shows the "not configured" message.
- [ ] `.env.example` documents the key without a value.
- [ ] Tests render the route inside a router and assert the visible menu and page.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Only `REACT_APP_*`, `NODE_ENV` and `PUBLIC_URL` reach the browser bundle at build time. Runtime values come through `window.GO1`.
- Never print or commit `.env.local`, JWTs or keys.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

None.
