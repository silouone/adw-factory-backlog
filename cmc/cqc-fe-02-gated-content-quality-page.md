---
id: cqc-fe-02-gated-content-quality-page
type: feat
status: blocked
priority: 1
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-02-gated-content-quality-page-1790563990079","branch":"adw/cqc-fe-02-gated-content-quality-page","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-02-gated-content-quality-page-1790563990079/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"}]
---
# A gated, empty Content quality page is reachable from the CMC menu

First slice of the Content quality page (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 1–4, "Page shape" and "Gating" sections). It establishes the
route, the menu item, the Statsig feature gate and the runtime configuration
every later ticket builds on.

## What to build

- A **Statsig feature gate** hook built on the Statsig legacy JS SDK, which the
  operator adds to `package.json` before this ticket is dispatched. Do not add
  or upgrade any dependency yourself. While the gate is loading, errors, or has
  no client key or gate id configured, it counts as **closed** (fail closed).
- **Runtime configuration** for three new keys: the CQC API base URL, the
  Statsig client key and the Statsig gate id. They follow the existing path:
  container env → nginx SSI → `window.GO1` → the app config module. Add them to
  `.env.example` with a comment and no real value. Also add them to any
  in-repo nginx SSI or k8s configmap template that already lists the
  existing keys.
- A **"Content quality" left-menu item** and a route in the flat route table,
  both wrapped by the gate. Gate closed: no menu item, and typing the URL shows
  the CMC's existing not-found page.
- The route renders a single root implementation from a new feature folder
  under `src/components/`. For now it shows the page header and, when the CQC
  base URL is not configured, a clear "Content Quality Checker API is not
  configured" message instead of content.

## Acceptance criteria

- [ ] Gate open: the menu item is visible and the route renders the Content quality page header.
- [ ] Gate closed, loading, or errored: no menu item, and the route renders not-found.
- [ ] Missing client key or gate id: the gate is closed and nothing throws.
- [ ] CQC base URL missing: the page shows the "not configured" message.
- [ ] `.env.example` documents the three keys without values.
- [ ] Tests render the route inside a router with the gate hook as a trivial mock (open/closed), and assert the visible menu and page, not hook calls.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and the conventions it routes to before editing.
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- Only `REACT_APP_*`, `NODE_ENV` and `PUBLIC_URL` reach the browser bundle at build time. Runtime values come through `window.GO1`.
- Never print or commit `.env.local`, JWTs or keys.

## Verify

The factory gates run tslint, jest and `npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

None. This ticket can start immediately, once the operator has added the Statsig dependency.
