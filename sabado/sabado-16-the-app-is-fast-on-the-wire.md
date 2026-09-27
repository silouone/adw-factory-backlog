---
id: sabado-16-the-app-is-fast-on-the-wire
type: feat
status: done
priority: 2
created: 2026-09-19
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-16-the-app-is-fast-on-the-wire-1789850372221","branch":"adw/sabado-16-the-app-is-fast-on-the-wire","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-16-the-app-is-fast-on-the-wire-1789850372221/workspace","outcome":"blocked"},{"runId":"sabado-16-the-app-is-fast-on-the-wire-1789859144755","branch":"adw/sabado-16-the-app-is-fast-on-the-wire","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-16-the-app-is-fast-on-the-wire-1789859144755/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1224"}]
---
# perf(front): assets are gzipped on the wire and the heavy pages load as their own chunks

> **Audit:** **P1-19** (axes K, L) — Lane D. Review stays on (nginx and the
> deploy scripts carry no test of their own).

## What happens today

- `frontend/src/App.tsx`: 32 routes, 32 static page imports, `lazy(` = 0.
  Every route — `/login` included — downloads Calendar, Assets, Admin and
  the 1 711-line SharedDocsPage before first paint.
- Eager JS = 1 497 432 B against the 1 500 000 cap at
  `frontend/src/bundle.test.ts:142` — **2 568 B of headroom**. The next
  front PR trips a test meant as a ratchet, and the predictable reaction is
  to raise the number.
- Live `curl -sI -H 'Accept-Encoding: gzip' https://app.sabado.io/assets/index-*.js`
  → `content-length: 1129470`, **no `content-encoding`**; same on staging.
  `grep -rn gzip deploy/nginx/` → nothing; `nginx:alpine` ships gzip off.
  Eager JS+CSS 1.68 MB raw vs 442 kB gzipped: every cold phone visit and
  every deploy re-download pays 4×.

## Requirements

- [ ] **R1 — gzip.** In both `server` blocks of `deploy/nginx/nginx.conf`
      and `deploy/nginx/nginx.prod.conf`: `gzip on; gzip_types text/css
      application/javascript application/json image/svg+xml
      application/manifest+json; gzip_min_length 1024;`. Nothing else in
      those files.
- [ ] **R2 — the deploy proves it.** In the post-health step of
      `deploy/staging-deploy.sh`, `deploy/slot-deploy.sh` and
      `deploy/production-deploy.sh`: `curl -sI -H 'Accept-Encoding: gzip'
      "$URL/assets/<main chunk>" | grep -qi '^content-encoding: gzip' ||
      fail`, resolving the main chunk from `dist/index.html`. `sabado-18`
      owns the pre-step-4 region of `production-deploy.sh`; this ticket owns
      only the post-health step.
- [ ] **R3 — lazy routes.** `React.lazy` + **one** `<Suspense>` around the
      route outlet for `AdminPage`, `CalendarPage`, `SharedDocsPage`,
      `AssetDetailPage` and the `creation/*` wizards. The fallback is an
      existing DS primitive (no new component unless none fits, and then
      one, in `ds/`). `ds:check` stays green.
- [ ] **R4 — the ratchet has teeth.** `bundle.test.ts`: keep the
      1 500 000 cap; add `eager total < 1 000 000`; add `lazy(` count in
      `App.tsx` ≥ non-auth routes − 3.
- [ ] **R5 — prod nginx parses before prod.** `nginx.prod.conf` is mounted
      only in prod (P1-13); before the PR is opened, both confs pass
      `docker run --rm -v "$PWD/deploy/nginx/<conf>:/etc/nginx/conf.d/default.conf:ro"
      nginx:alpine nginx -t` (the TLS `include`/cert lines may need a stub
      mount — say so in the PR body if they do).

## Files

`deploy/nginx/nginx.conf` · `deploy/nginx/nginx.prod.conf` ·
`deploy/staging-deploy.sh` · `deploy/slot-deploy.sh` ·
`deploy/production-deploy.sh` (post-health step only) ·
`frontend/src/App.tsx` · `frontend/src/bundle.test.ts`.

## Verify

- [ ] Red test: `bundle.test.ts` "eager total < 1 000 000" — RED today
      (1 497 432).
- [ ] Red test: `bundle.test.ts` "`lazy(` count ≥ non-auth routes − 3" —
      RED today (0).
- [ ] `npm run build` report: `AdminPage`, `CalendarPage`, `SharedDocsPage`,
      `AssetDetailPage` and each `creation/*` wizard appear as their own
      chunk; eager JS total printed in the PR body.
- [ ] `nginx -t` in `nginx:alpine` → `syntax is ok` for both confs.
- [ ] `bash -n` on the three deploy scripts.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `just test` — all green.
- [ ] **Operator, after merge:** staging serves `content-encoding: gzip` on
      the main chunk (the R2 check prints it); promote only after that.

## Out of scope

Security headers (P1-8, held); `vite-plugin-compression`; the font precache
(L P3); `proxy_read_timeout`; `deploy.previous` rollback (P1-13).
