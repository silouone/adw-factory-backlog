---
id: adw-fe-05-web-grid
type: feat
status: done
priority: 2
created: 2026-09-12
depends: [adw-fe-01-journal-schema]
caps: {minutes: 120, turns: 600}
attempts: [{"runId":"adw-fe-05-web-grid-1789315506448","branch":"adw/adw-fe-05-web-grid","workspace":"/Users/silouane/adw-factory/runs/adw-fe-05-web-grid-1789315506448/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/15","provider":"claude","model":"sonnet"}]
---
# `adw web` — the run grid over the banked journals

> Part of the v1.2 live view. **Spec: `specs/adw-v1.2-live-view.md`** — read it
> before starting; it carries the decisions and the reasoning, this ticket is
> only the work order. Decomposed 2026-09-12.
>
> **`depends:` is NOT enforced by the factory** — it is parsed by nobody
> (`grep depends src/intake/` → 0 hits). It is a note to the operator and to
> you. Check the blockers really are `done` before starting.

## Context

**The first thing you can look at.** Needs only the schema ticket: it renders
the 67 journals already on disk. Everything after this thickens it.

This ticket establishes the two new seams for the whole view, and the split is
mandatory rather than stylistic (Art. IX): a **pure projection** holding every
rendering rule, and a **thin server edge** with its filesystem, watch and clock
dependencies injected like every other edge in this codebase.

## Requirements

- [ ] A **pure** projection: `(journal records) → run view`. No I/O. Run id,
      ticket id, target, lane, isolation, provider, outcome, duration.
- [ ] `adw web` subcommand serving a grid, newest first, one card per run.
- [ ] **Zero new runtime dependencies** — `Bun.serve` and SSE ship with the
      runtime. Two screens of positioned rectangles do not need a framework,
      and every dependency added here is one in the factory's manifest forever
      (Art. VIII).
- [ ] **Loopback bind + a per-launch token** printed to the terminal.
- [ ] **Read-only by construction**: no mutating route exists. Not a permission
      check that can be misconfigured — the capability is absent.
- [ ] A truncated or in-flight journal renders up to its last complete line; a
      run whose journal is missing is skipped with a notice, never a crash.
- [ ] A `just watch` recipe wrapping the subcommand.
- [ ] `test/cli-surface.test.ts` updated — this widens the tested CLI surface.

## Verify

- [ ] Projection tests are **plain arrays in, view out** — no server, no
      browser. **The browser is never a test seam**; if a behaviour cannot be
      asserted at the projection it is presentation and is not specified.
- [ ] A request without the token is refused; no mutating route exists.
- [ ] A truncated final line is ignored and everything before it renders.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
