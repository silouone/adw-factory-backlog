---
id: adw-render-01-foundation
type: feat
status: done
priority: 1
created: 2026-09-18
review: true
caps: {minutes: 180, turns: 900}
depends: []
attempts: [{"runId":"adw-render-01-foundation-1789771519506","branch":"adw/adw-render-01-foundation-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-render-01-foundation-1789771519506/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# P0 — the Preact foundation: build, test seam, bundle route, and one reference component

> **Spec authority:** `specs/adw-v1.11-render-architecture.md` §4 **P0** and
> §6 **D2**/**D3**. Stack decided by `specs/adw-v1.5-operator-console.md` §3
> **D1** (operator, 2026-09-18): Preact + `@preact/signals`.

## Rescoped 2026-09-18 — from "invent the pattern" to "execute a fixed one"

This ticket was `type: manual` because P0 is where the conventions get decided,
and an agent **inventing** them unattended would have P1–P4 inherit whatever it
happened to choose. That reasoning stands. The fix is to remove the invention,
not to ignore it.

Every open convention is now **decided below** — component directory, test
directory, bundle entrypoint, shared CSS module, and the exact reference
component. What remains is transcription against a fixed pattern, which is
dispatchable. **Where this ticket names a path or a file, use that path.** If a
decision here proves unbuildable, stop and propose a spec amendment — do not
substitute your own choice, because P1–P4 inherit it.

`review: false` is deliberate: the eight requirements below carry hard machine
bounds (lint/tsc/test green, `/bundle.js` 200/401, the parity suite still
green, nothing visual moves). The gates *are* the review here. Recorded in the
journal as an explicit opt-out, never a silent skip.

## Decisions — these are fixed, not suggestions

| | |
|---|---|
| **Components live in** | `src/web/ui/` — one file per component, `.tsx` |
| **Component tests live in** | `test/web/ui/` — **not** beside the component |
| **Bundle entrypoint** | `src/web/ui/entry.tsx` |
| **Shared CSS module** | `src/web/ui/css.ts`, exporting `css(): string` |
| **Reference component** | `renderBar` (`src/web/render.ts:235`) → `<Bar/>` |

### Why tests cannot be colocated — read this before writing one

`bunfig.toml` sets `[test] root = "test"`. A test written at
`src/web/ui/Bar.test.tsx` **is never executed**, and `bun test` still reports
green. That is a silent pass, not a pass.

**Check, do not assume:** record `bun test`'s test count before adding the
first component test and after. The count must go **up**. If it did not, the
file is in the wrong place.

### Why `renderBar` is the reference component

It is the only genuine leaf in the web layer, confirmed 2026-09-18:

- **One call site** — `src/web/render.ts:270`, inside `renderMiniChart`.
- **Zero direct test references** — nothing asserts its output in isolation, so
  the component test you write is the first, and it goes red cleanly.
- **Pure function of one typed prop** — `CardLaneBar`, exported from
  `src/web/card.ts:35`, already derived. No new derivation logic is needed,
  which is exactly the property P1–P4 must copy.
- **Board-only** — it carries no cross-screen parity contract.

> **Do not use `renderUsageChip`.** An earlier draft suggested it. It has three
> call sites, is serialized into a `window.__upChip` seed
> (`render.ts:861`, `render-run.ts:428`), is returned as a string by
> `GET /usage.json` (`server.ts:322`), and is pinned by ~20 assertions in
> `test/web/render.test.ts` plus byte-identity in `test/web/usage-parity.test.ts`.
> Converting it breaks three contracts at once.

## Requirements

- [ ] **R1 — dependencies.** `preact` + `@preact/signals` as `dependencies`;
      `@testing-library/preact` + `happy-dom` (or
      `@happy-dom/global-registrator`) as `devDependencies`. **No other
      runtime dependency.** Measured baseline to stay near: 3 runtime
      packages, 8.1 KB gzipped.
- [ ] **R2 — TypeScript config.** `tsconfig.json` gains exactly
      `"jsx": "react-jsx"` and `"jsxImportSource": "preact"`. Every existing
      strict flag stays — in particular `exactOptionalPropertyTypes` and
      `noUncheckedIndexedAccess`, both **verified 2026-09-18 to enforce
      through JSX props** (a `<Card elapsedMs={undefined}/>` errors `TS2375`).
      Weakening any strict flag to make a component compile is a blocker,
      not a fix.
- [ ] **R3 — the test seam.** `bun test` runs component tests with no second
      test runner. Register happy-dom via `bunfig.toml`'s `[test] preload`,
      keeping `root = "test"` as it is. Component tests go in `test/web/ui/`.
      **Verified 2026-09-18:** 2 component tests in 240 ms, and breaking the
      component produces a real red diff. If this requires vitest or jest,
      stop — that contradicts §6 D2's "no second toolchain" and needs a spec
      amendment.
- [ ] **R4 — the bundle, built at launch and served from memory.**
      `Bun.build({entrypoints: ["src/web/ui/entry.tsx"], target: "browser",
      minify: true})` called programmatically inside the web server; serve
      `.outputs[0].text()` from a new **`GET /bundle.js`**, token-gated like
      every other route. Add the branch beside the existing ones in
      `src/web/server.ts` (the `/usage.json` branch at line 245 is the shape
      to copy — the token check above it already covers the new route).
      **Verified 2026-09-18:** 11 KB in 3 ms, servable from memory. No build
      artifact is committed; no watch mode; no dev server.
- [ ] **R5 — the read-only guarantee is preserved.** `test/web/server.test.ts`
      asserts structurally that `src/web/server.ts` contains no `"POST"`,
      `"PUT"`, `"DELETE"` or `"PATCH"` literal. A `GET` branch does not trip
      it; **confirm the test still passes rather than assuming.**
- [ ] **R6 — the CSS ports verbatim (§6 D3).** The 472 lines across
      `render.ts:1093`'s and `render-run.ts:835`'s `css()` move unchanged into
      `src/web/ui/css.ts`. Both existing call sites import from there. **No
      restyle — not one selector, not one value.** The design is what v1.7/v1.10
      specify; changing it here would silently amend two specs.
- [ ] **R7 — one reference component, end-to-end.** Add
      `src/web/ui/Bar.tsx` exporting `<Bar/>`: a pure function of one
      `CardLaneBar` prop, producing the same markup `renderBar` produces today.
      Its test is `test/web/ui/Bar.test.tsx`, asserting rendered output via
      `@testing-library/preact` — never regex over an HTML string (§6 D4).
      Props come from `card.ts`; **no new derivation logic**.

      **`renderBar` stays exactly as it is, and stays the server renderer.**
      P0 does not convert a screen (§4 — P2 does the board), so `<Bar/>` is
      not yet mounted by any page. It is the reference P1–P4 copy, and its
      component test is what proves it. `src/web/ui/entry.tsx` therefore
      mounts a minimal root that imports `<Bar/>` so the bundle has a real
      entry to build — nothing more. Do not add `preact-render-to-string`;
      no server path needs it in P0.
- [ ] **R8 — Article IX, written down.** A short `src/web/ui/README.md`
      stating the rule §7 of the spec sets: a component is a pure function of
      its props; it may not read the clock, the filesystem or the network; the
      single `now` signal at the root is the only clock reader. P1–P4 will be
      reviewed against this.

## Order of work — Article I

1. Write `test/web/ui/Bar.test.tsx` against the not-yet-existing `<Bar/>`.
2. Confirm it is **red**, and that `bun test`'s count went **up** (R3).
3. Only then build R1–R8 to green.

## Verify

- [ ] `bun run lint && bunx tsc --noEmit && bun test` — green.
- [ ] `bun test`'s test count is **higher** than before this ticket, and the
      increase is the component test(s) — proof they actually ran.
- [ ] `just web` serves `/bundle.js` (200, token-gated; 401 without a token).
- [ ] The reference component's test goes **red** when `<Bar/>` is broken and
      green when restored — pasted into the ticket as evidence, per Art. I.
- [ ] `test/web/server.test.ts`'s no-mutating-route tests still pass.
- [ ] `test/web/usage-parity.test.ts` and `test/web/render.test.ts` pass
      **unmodified** — neither is in scope.
- [ ] Both screens still render exactly as before (R6 — nothing visual moves).

## Out of scope

- Converting any screen. P2/P3 do that.
- Changing the SSE payload. That is P1 (`adw-render-02`), deliberately
  separate so the contract change is provable on its own.
- Any redesign (§8 non-goals).
- A second test runner. `bun test` is the only runner (R3).
- Editing `test/web/usage-parity.test.ts` or `test/web/render.test.ts`. If a
  change here would require touching either, the approach is wrong — stop.
- Touching `renderUsageChip` / `renderUsagePanel` in any way.
