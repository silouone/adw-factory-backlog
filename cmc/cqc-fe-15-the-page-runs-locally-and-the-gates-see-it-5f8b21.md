---
id: cqc-fe-15-the-page-runs-locally-and-the-gates-see-it-5f8b21
type: chore
status: in-progress
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
caps: {minutes: 120, turns: 500, stallMinutes: 25}
depends: []
attempts: []
---
# A developer can run the Content quality page locally, and the gates see what the dev server sees

Three defects in the local-development path, all found on 2026-09-29 while mounting the page
against the deployed CQC dev API for the first time. Each one costs a developer time or risks
a leaked credential, and none is visible to the current gates. Evidence:
adw-factory `ai_docs/2026-09-29-cqc-dev-integration-findings.md`.

## Base

**Base branch: `cqc/release-1`**, not `master`. Branch from it and target it; the operator
merges there. Never target `master`.

## 1. The gates cannot see the dev server's tsconfig

`npm run build` type-checks with `tsconfig.prod.json`, which extends `tsconfig.json` and sets
`noImplicitAny: false`. The dev server (`config/webpack.config.dev.js`, `ForkTsCheckerWebpackPlugin`
with `paths.appTsConfig`) uses the **strict** `tsconfig.json`. So code can pass lint, jest and
build and still refuse to compile under `npm start` — which is exactly what happened in
`cqc-fe-13-dev-server-type-error-4e1c77` (PR #147).

Add a script that type-checks against `tsconfig.json` — `tsc --noEmit -p tsconfig.json` — and
make it part of what the repo considers green. Do not change either tsconfig's strictness:
that is a larger decision and would touch the whole codebase. Only make the stricter check
runnable and visible.

## 2. `PORT=3000 npm start` silently does nothing

`package.json` has `"start": "PORT=3005 node scripts/start.js"`. The inline assignment beats
the environment, so the dev server always binds 3005 no matter what the caller sets. The CQC
backend's `dev` CORS allowlist is `https://staff-new.qa.go1.cloud` and `http://localhost:3000`
(BE-15), so a developer following the obvious recipe gets an opaque CORS failure with nothing
pointing at the port.

Make `PORT` overridable while keeping 3005 as the default, e.g.
`"start": "PORT=${PORT:-3005} node scripts/start.js"`. Verify both
`npm start` (3005) and `PORT=3000 npm start` (3000).

## 3. `.env` is not gitignored, and it is where the JWT goes

`.gitignore` covers `.env.local`, `.env.development.local`, `.env.test.local` and
`.env.production.local` — but **not** `.env`, which is the file `.env.example` invites you to
copy and fill with `REACT_APP_JWT` and `CMC_STAFF_COOKIE`. One `git add -A` publishes a live
staff token.

Add `.env` to `.gitignore`, and update `.env.example`'s header to say that local secrets belong
in `.env.development.local` (gitignored, and CRA loads it ahead of `.env`). Do not commit or
delete anyone's existing `.env`.

## Acceptance criteria

- [ ] A script type-checks against `tsconfig.json`, and it is green on `cqc/release-1` today.
- [ ] `PORT=3000 npm start` binds 3000; bare `npm start` still binds 3005.
- [ ] `.env` is gitignored; `git check-ignore .env` exits 0.
- [ ] `.env.example` says where secrets go and why.
- [ ] tslint, jest and build stay green — this ticket adds a check, it does not change app code.

## Also worth doing, if it is cheap

`README` or `.env.example` gains the working local recipe: `REACT_APP_CQC_API_URL` pointing at
the CQC dev API, `REACT_APP_JWT` obtained from
`GET https://staff.qa.go1.cloud/api/user/account/current` with the staff cookie (the response's
`jwt` field), and `PORT=3000`. Skip it if it fights the repo's existing docs.

## Blocked by

- (nothing — this is independent of the CQC backend work)
