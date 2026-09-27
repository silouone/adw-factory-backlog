---
id: adw-gates-04-the-documented-verify-command-is-not-the-gate
type: chore
status: done
priority: 1
review: false
created: 2026-09-19
caps: {minutes: 60, turns: 300}
depends: []
attempts: [{"runId":"adw-gates-04-the-documented-verify-command-is-not-the-gate-1789893089942","branch":"adw/adw-gates-04-the-documented-verify-command-is-not-the-gate","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-gates-04-the-documented-verify-command-is-not-the-gate-1789893089942/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/94","provider":"claude","model":"sonnet"}]
---
# `just verify` runs a command that fails 170 tests the real gate passes

> Found 2026-09-19 while salvaging
> `adw-render-03-the-board-becomes-components-1789846487294`.

## The defect

`package.json` defines the test script as:

```
"test": "bun test --timeout=30000 --preload ./test/setup/happy-dom.ts"
```

The `adw-factory` target's `test` gate runs **`bun run test`** — it gets the
preload, and it passes.

But the definition of done documented in three places runs **bare
`bun test`**, which skips the preload:

- `justfile` — `verify: bun run lint && bunx tsc --noEmit && bun test`
- `README.md:133`
- **`CLAUDE.md:23`** — "Verify before done"

Without `test/setup/happy-dom.ts`, every DOM-touching test throws
`ReferenceError: document is not defined`. Measured in a gate-green
workspace: **2,648 pass, 170 fail, 7 errors** — none of the failures real.

## Why it matters more than a typo

`CLAUDE.md` is listed in `targets/adw-factory.json`'s `context:` array. Every
agent the factory dispatches **against itself** is handed this command as its
definition of done. An agent that dutifully runs the documented verify sees
170 failures it did not cause, and its available responses are all bad: chase
phantoms, weaken real tests, or report blocked on a green tree.

The defect has existed since `adw-render-01` added the first `.tsx` and the
`happy-dom` preload alongside it. It is latent until a run touches the view
layer — which is every remaining ticket in the v1.11 migration (P2, P3, P4).

## Requirements

- [ ] **R1 — one command, one definition.** `just verify`, `README.md` and
      `CLAUDE.md` all invoke `bun run test`, matching the gate.
- [ ] **R2 — `just test` too.** Any other recipe invoking bare `bun test`
      gets the same treatment; `grep -n "bun test" justfile` is the sweep.
- [ ] **R3 — no new script.** Do not add a second test entry point to
      `package.json`. The fix is to call the one that exists.
- [ ] **R4 — `test-fast` keeps its scope.** `test:unit` is a deliberate
      deterministic subset; if it needs the preload to stay honest, add it
      there too, but do not widen what it runs.

## Verify

- [ ] `grep -rn "bun test" justfile README.md CLAUDE.md` returns no bare
      invocation.
- [ ] `just verify` in a clean checkout is green, and its test count matches
      `bun run test`'s.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` — green.

## Out of scope

- Making the full suite deterministic. README's "Where it can still fail"
  documents real container/E2B flake; this ticket is only about the 170
  failures that are pure configuration.
