---
id: sabado-27-the-built-front-can-talk-to-a-real-backend-acabfa
type: chore
status: in-review
priority: 1
created: 2026-09-28
caps: {minutes: 120, turns: 400, stallMinutes: 25}
depends: []
attempts: [{"runId":"sabado-27-the-built-front-can-talk-to-a-real-backend-acabfa-1790593594157","branch":"adw/sabado-27-the-built-front-can-talk-to-a-real-backend-acabfa","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-27-the-built-front-can-talk-to-a-real-backend-acabfa-1790593594157/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1311","provider":"claude","model":"sonnet"}]
---
# test(e2e): the built front can talk to a real backend, so a journey can be written at all

> **Why this is its own ticket.** `sabado-25` landed Playwright against
> `vite preview` — and `vite preview` has **no `/api` proxy**.
> `frontend/vite.config.js:161-168` declares `proxy` under **`server:`** only
> (the dev server, :5173); there is no `preview:` block. That is why
> `boot.spec.ts` stubs `page.route('**/api/**')`: it had no other option. Until
> the preview server can reach a backend, **no real e2e journey is writable**,
> and every journey ticket behind this one is blocked on ~8 lines of config.

## What happens today

One Playwright project, one spec, API stubbed:

- `frontend/playwright.config.ts` — `projects: [{ name: 'chromium' }]`,
  `webServer: npm run preview -- --port 4173`, `baseURL` hardcoded to
  `http://localhost:4173`.
- `frontend/e2e/boot.spec.ts` — 7 **pre-auth** routes, `stubApi()` fulfilling
  every `**/api/**` with `{}`. It tests **the bundle**, deliberately with no
  backend: it is fast, it needs no services, and it is the factory gate
  `test-e2e-boot` (`adw-factory targets/sabado.json`).

`App.tsx` declares **33 routes**. The 26 authenticated ones
(`/dashboard`, `/calendar`, `/documents`, `/vault`, `/foyer`, `/biens`,
`/circles`, `/children`, `/budget`, `/sabado`, `/contacts`, `/settings`,
`/biens/nouveau/:track`, `/foyer/nouveau/:track?`, …) are never visited in a
browser. The lazy-chunk test `import()`s their chunks — that catches a cyclic
TDZ crash, it never mounts a page.

## Requirements

- [ ] **R1 — `preview.proxy`.** `frontend/vite.config.js` gains a `preview:`
      block whose `proxy` mirrors the existing `server.proxy` exactly, target
      from **`E2E_API_TARGET`** (falling back to `server`'s
      `VITE_API_TARGET || 'http://localhost:8010'`), with the same
      `stripApiPrefix` rule — strip `/api` for a bare uvicorn, keep it behind an
      nginx. Verified supported: `PreviewOptions extends CommonServerOptions`,
      which declares `proxy?: Record<string, string | ProxyOptions>`
      (`node_modules/vite/dist/node/index.d.ts:763` → `:721`, vite 6.4.3).
      Factor the proxy object into one `const` used by both blocks — two copies
      drift.
- [ ] **R2 — two Playwright projects, and the switch is a conditional spread.**
      `playwright.config.ts` splits into:
      - **`bundle`** — `testDir: './e2e/bundle'`, holding today's
        `boot.spec.ts` moved there unchanged. Keeps its stub. No backend.
      - **`journeys`** — `testDir: './e2e/journeys'`, `baseURL` from
        `E2E_BASE_URL` (default `http://localhost:4173`).

      `npm run test:e2e` is `(test -d dist || npm run build) && playwright test`
      — **bare `playwright test` runs every project in the array**, so an unset
      env var does not exclude anything on its own. The `projects` array must be
      built conditionally:

      ```js
      projects: [
        bundle,
        ...(process.env.E2E_API_TARGET ? [journeys] : []),
      ]
      ```

      Without that spread the `journeys` project runs against no backend and the
      factory gate goes red on every sabado ticket. This is the mechanism R3
      depends on.
- [ ] **R3 — the factory gate does not change behaviour.** `test-e2e-boot` in
      `adw-factory targets/sabado.json` is `cd frontend && npm run test:e2e` and
      sets no `E2E_API_TARGET`: it must keep running the `bundle` project alone
      and stay green. **This is the requirement that stops this ticket turning
      every later sabado run red.** `npm run test:e2e` keeps its current
      meaning; a second script `test:e2e:journeys` is *not* added — the projects
      and the env var are the whole switch.
- [ ] **R4 — one journey proves the wire, and it is deleted.** A throwaway spec
      under `e2e/journeys/` hits `GET /api/health` **through the proxy** and
      asserts the real payload's `commit` field — something `stubApi`'s `{}`
      cannot fake. It exists to verify R1 and is **removed in the same PR**
      after the Verify step below records it passing; `sabado-29` writes the
      journeys that stay. Say so in its own docstring so nobody keeps it.
- [ ] **R5 — `just check` and CI keep their current cost.** Neither sets
      `E2E_API_TARGET` in this ticket (that is `sabado-29`), so both keep
      running `bundle` only and neither gets slower.

## Files

`frontend/vite.config.js` (R1) · `frontend/playwright.config.ts` (R2) ·
`frontend/e2e/boot.spec.ts` → `frontend/e2e/bundle/boot.spec.ts` (moved, content
unchanged — the `DIST` path constant walks `..` and must be re-checked) ·
`frontend/e2e/journeys/00-wire.spec.ts` (new, deleted in the same PR).

## Verify

- [ ] `cd frontend && npm run test:e2e` with `E2E_API_TARGET` **unset** →
      the `bundle` project only, same test count as before this ticket, green.
- [ ] With a backend up on :8010 and `E2E_API_TARGET=http://localhost:8010`,
      the R4 wire spec reads a real `commit` from `/api/health`. Record the
      value in the PR body, then delete the spec.
- [ ] `grep -rn "page.route" frontend/e2e/journeys/` → no match (journeys never
      stub; that is what makes them e2e).
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `test-e2e-boot` · `just test` — green.

## Out of scope

The CI job, any backend boot, the account fixture, and every real journey —
`sabado-28` … `sabado-31`. This ticket only makes them possible.
