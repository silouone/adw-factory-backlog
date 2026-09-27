---
id: cqc-fe-01-drawer-keeps-keyboard-focus
type: feat
status: blocked
priority: 1
created: 2026-09-26
model: gpt-5.6-sol
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: []
attempts: [{"runId":"cqc-fe-01-drawer-keeps-keyboard-focus-1790532937437","branch":"adw/cqc-fe-01-drawer-keeps-keyboard-focus-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/cqc-fe-01-drawer-keeps-keyboard-focus-1790532937437/workspace","outcome":"blocked","provider":"codex","model":"gpt-5.6-sol"}]
---
# The CMC Drawer keeps keyboard focus where a keyboard user expects it

Prefactor for the Content quality page (spec `docs/cqc/spec-cqc-fe-release-1.md`,
user stories 30 and 55, "Components" section). The page's run drawer reuses the
CMC's own `Drawer` component at size md (900px). Today that drawer does not
manage focus, so a keyboard or screen-reader user loses their place when it
opens, changes content or closes.

## What to build

Extend the existing CMC `Drawer` so that, for every consumer:

- when it opens, focus moves to its heading;
- when its content changes to another item while open (a caller-supplied key
  changes), focus moves to the heading again;
- while open, Tab and Shift+Tab stay inside the drawer (focus trap);
- when it closes (close button, Escape, backdrop click), focus returns to the
  element that opened it;
- Escape closes it even when focus is inside a text input: the Escape handler
  runs before any input guard.

The new behaviour is opt-in or backward compatible: the existing drawers
(retirement request, partner agreement, preview) keep their current
behaviour and their tests stay green.

## Acceptance criteria

- [ ] Opening the drawer from a button puts focus on the drawer heading.
- [ ] Changing the drawer's content key while it is open moves focus to the heading.
- [ ] Tab from the last focusable element wraps to the first; Shift+Tab from the first wraps to the last.
- [ ] Escape, the close button and a backdrop click each close the drawer and return focus to the opener.
- [ ] Escape pressed inside an `<input>` in the drawer closes it.
- [ ] The drawer exposes a dialog role with an accessible name taken from its heading.
- [ ] Existing Drawer consumers' tests pass unchanged.
- [ ] Colocated React Testing Library tests drive each behaviour with real key presses and assert `document.activeElement` and roles.

## Constraints

- Tests and the build run only in the factory gates (Docker, Node 14). The host has Node 25 and the sandbox has no network, so do not run `npm test`, `npm run build` or `pnpm` locally. Do not edit `package.json`, the lockfile, or the jest, webpack or tsconfig configuration.
- Read `AGENTS.md` and the conventions it routes to before editing (React rules,
  `references/testing.md`, `verification.md`).
- TypeScript 3.5 and React 16.8: no `?.`, no `??`, no `Array.flat`/`flatMap`, no `Promise.allSettled`.
- No new dependency.

## Verify

The factory gates run tslint on `src`, the jest suite and
`npm run build` (which type-checks) in Docker `node:14.21.3`.

## Blocked by

None. This ticket can start immediately.
