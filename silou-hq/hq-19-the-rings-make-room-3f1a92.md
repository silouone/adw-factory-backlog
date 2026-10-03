---
id: hq-19-the-rings-make-room-3f1a92
type: feat
status: done
priority: 1
depends: []
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-19-the-rings-make-room-3f1a92-1790981272143","branch":"adw/hq-19-the-rings-make-room-3f1a92","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-19-the-rings-make-room-3f1a92-1790981272143/workspace","outcome":"in-review","provider":"claude","model":"claude-sonnet-5-5","pr":"https://github.com/silouone/silou-hq/pull/16"}]
---
# The rings make room: runtimes only, a plugins track, docked instructions, a real clock

> Spec: `~/personal_project/silou-hq/docs/spec-v2.md` (binding; it amends `docs/spec-v1.md`). Rules: `CLAUDE.md`. Writes go only through `POST /action` (amended rule #1). Reference implementation for feel: branch `proto/action-ring` @ `5eecf24` in `~/personal_project/silou-hq-proto-ring` (never merged; read, do not copy wholesale). Its `PROTOTYPE-NOTES.md` holds every constant.

## What to build

The landing's rings are reorganised (spec-v2 → Landing changes; stories 1–8):
- The inner ring is LLM runtimes only (claude code, codex, ollama), drawn as a real band, titled `RUNTIMES · ANY LLM`.
- Plugins (squares) and MCP servers (diamonds) move to their own track at `R_EXT = 0.141` U in `#7FA7B5`, titled `PLUGINS · MCP · n`. A plugin whose label equals a runtime's label reads `<label> · plugin`.
- Each runtime's instruction file (CLAUDE.md, AGENTS.md) docks beside its runtime (angle +0.3 rad, radius R.rt+0.03). A runtime whose instruction file is missing shows it as missing (today: codex has no `~/.codex/AGENTS.md`).
- The hooks and live-sessions rings become visible, titled bands with counts.
- The routines ring is a 24 h clock: 96 ticks, numerals every 3 h, a today-so-far arc, an ember `NOW HH:MM` hand.
- Always-on routines move to their own arc at 6 o'clock, at most 40°, evenly spaced.
- The `band()` helper draws with a `moveTo` between arcs (no wedge).
- **The centre is left empty** (ASK HQ comes in hq-30).

## Red first

Pure ring-layout tests (beside hq-09's):
- the always-on arc: angles centred on 6 o'clock, span ≤ 40°, evenly spaced, stable for n = 0, 1, 5;
- the plugins·mcp track radius sits strictly between runtimes and skills;
- instruction docking: position per runtime, and a "missing" mark when the graph has no instruction node for it;
- the label rule: a plugin equal to a runtime label gets ` · plugin`;
- clock tick lengths for 00:00, 03:00, 01:00 and 00:15.

## Acceptance criteria

- [ ] No runtime ring position is taken by a plugin or an MCP server.
- [ ] Nothing is drawn at the rings' centre.
- [ ] Graph build test: a fixture home without `~/.codex/AGENTS.md` yields the missing mark for codex.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] At the reference display, labels on the always-on arc don't overlap when the routines ring is zoomed.

## Blocked by

- (nothing). The spec-v2 commit must be on main before dispatch.
