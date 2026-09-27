---
id: sabado-25-the-built-front-boots-in-a-browser-before-it-ships
type: feat
status: in-progress
priority: 1
created: 2026-09-20
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: []
---
# test(front): the built bundle boots in a real browser, in CI and as a factory gate, so a blank page never reaches staging again

> **Incident, 2026-09-20 15:26–15:55.** `#1230` (sabado-21) made
> `VaultContext` import `stores/vaultCountdown`, which already imported
> `AUTO_LOCK_MS` back from `VaultContext`. Rollup evaluated the store first,
> `Math.round(AUTO_LOCK_MS / 1000)` hit a `const` in its temporal dead zone,
> and the ENTRY chunk threw `ReferenceError: Cannot access 'Fe' before
> initialization` — a blank page on every route of staging, HTTP 200
> everywhere, zero 5xx in nginx. Fixed in `#1234`. **Every gate was green:**
> 3 450 vitest tests (vite-node evaluates modules on demand, in another
> order), `npm run build` (a build does not execute), five deploy proofs
> (`/api/health` answers, the main chunk is gzipped). Nothing in the repo
> loads the built bundle in a browser. Audit axis H: "no e2e".

## What happens today

`frontend/dist/` is produced by `npm run build` and copied to the VM. The
first thing that ever executes it is a user's browser. `App.tsx` now has 24
`lazy()` pages (sabado-16), so the same class of failure can also hide in a
lazy chunk and appear only on the route that imports it.

## Requirements

- [ ] **R1 — the boot check.** `frontend/e2e/boot.spec.ts` under
      `@playwright/test` (chromium only), run against `vite preview` of the
      built `dist/`: for each pre-auth route (`/welcome`, `/login`,
      `/login/oublie`, `/register`, `/cgu`, `/confidentialite`,
      `/mentions-legales`), the page reaches a non-empty `#root`, no console
      error, no failed request. Then, in one page, `import()` every
      `dist/assets/*.js` chunk that `index.html` does not preload, and assert
      each evaluates — this is what catches a cycle inside a lazy chunk
      without logging in.
- [ ] **R2 — the entry chunk is asserted directly.** One test re-imports the
      entry chunk named in `dist/index.html` and asserts it resolves; a
      failed evaluation is cached as a rejection, so this is the smallest
      possible reproduction of the incident.
- [ ] **R3 — one command.** `npm run test:e2e` builds if `dist/` is absent,
      starts `vite preview` on a free port, runs the spec, stops the server.
      Chromium is installed by `npx playwright install chromium` in
      `scripts/bootstrap` (dev) and in the CI job (`--with-deps`).
- [ ] **R4 — wired where checks run.** `ci.yml` `front` job: a step after
      `npm run build`. `targets/sabado.json` (adw-factory) gains a gate
      `test-e2e-boot` between `frontend-build` and `test`. `just check` runs
      it. Budget: under 60 s on CI.
- [ ] **R5 — the cycle cannot come back.** `bundle.test.ts` gains a static
      check that no module under `src/stores/` imports from `src/contexts/`
      (the direction sabado-21's fix established), so the specific incident
      is also pinned without a browser.

## Files

`frontend/e2e/boot.spec.ts` (new) · `frontend/playwright.config.ts` (new) ·
`frontend/package.json` (`@playwright/test` devDependency, `test:e2e`) ·
`frontend/src/bundle.test.ts` · `scripts/bootstrap` · `.github/workflows/ci.yml`
· `justfile` · adw-factory `targets/sabado.json` (operator, after merge).

## Verify

- [ ] Red test: check out `d3cfa2bc` (main before `#1234`), build, run
      `npm run test:e2e` → the entry-chunk test fails naming
      `Cannot access 'Fe' before initialization`. On `main` → green.
- [ ] Red test: `bundle.test.ts`'s direction check fails on `d3cfa2bc`'s
      `stores/vaultCountdown.ts`, passes on `main`.
- [ ] Every lazy chunk (24 pages) evaluates; the test names the chunk on
      failure.
- [ ] `ci.yml` `front` job green with the new step; wall time printed in the
      PR body.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.

## Out of scope

Logged-in journeys, account creation, anything that needs a slot or a
secret — that is `sabado-26`. Firefox/WebKit. Visual regression.
