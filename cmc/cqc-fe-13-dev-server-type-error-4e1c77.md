---
id: cqc-fe-13-dev-server-type-error-4e1c77
type: bug
status: done
priority: 1
created: 2026-09-29
model: gpt-5.6-sol
depends: []
attempts: []
---
# The Content quality route did not compile under the dev server's tsconfig

**Record of work already merged (PR #147, 2026-09-29).** Built by hand during the first
local mount of the page, not dispatched. Kept here so the ledger matches the branch.

`npm start` failed to compile on `cqc/release-1`:

```
src/app/index.tsx(265,31)
TS7006: Parameter 'routeProps' implicitly has an 'any' type.
```

The dev server type-checks with `tsconfig.json` (`noImplicitAny: true`); `npm run build`
uses `tsconfig.prod.json`, which extends it and sets `noImplicitAny: false`. The target's
gates are lint + jest + build, so **no gate exercises the strict config** and the error
reached the branch. Anyone on CLB typing `npm start` would have hit it first.

Fixed by typing the `/quality` route's render prop with the repo's own
`RouteComponentProps<any>` from `src/utils/route` — the idiom already used in
`src/app/Portal/index.tsx`. That type carries the local `History`, which is what
`<ContentQuality history=...>` expects.

Verified on `node:14.21.3`: tslint clean, 952 tests in 121 suites, `npm run build`
compiled, and `node scripts/start.js` reaching `Compiled successfully!`.

**The gap is still open:** the gate set cannot see `tsconfig.json`. `cqc-fe-15-the-page-runs-locally-and-the-gates-see-it-5f8b21` closes it.
