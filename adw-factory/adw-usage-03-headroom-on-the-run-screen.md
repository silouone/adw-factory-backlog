---
id: adw-usage-03-headroom-on-the-run-screen
type: feat
status: done
priority: 2
created: 2026-09-17
depends: [adw-usage-02-subscription-headroom-on-the-board]
attempts: [{"runId":"adw-usage-03-headroom-on-the-run-screen-1789633166424","branch":"adw/adw-usage-03-headroom-on-the-run-screen","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-03-headroom-on-the-run-screen-1789633166424/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/66","provider":"claude","model":"sonnet"}]
---
# The run screen is where you wait, and it is the screen with no headroom

> Spec: `specs/adw-v1.7-subscription-headroom.md`, story 19. Depends on
> `adw-usage-02` landing the chips, panel, parser and probe — this ticket
> **reuses** them and adds no new data path.

## 1. Why this is not just "do it twice"

`adw-usage-02` puts the KPIs on the board, which is where you **dispatch**.
The run screen is where you **wait** — for 11 minutes on a median green run,
longer on a feat. That is exactly the window in which "have I got enough left
for the next one" gets asked, and it is the screen that cannot answer it.

The operator's ask was *"so it's always there"*. One screen is not always.

## 2. What lands

`renderRunHeader` carries the same two chips and the same panel as the board
header. Same component, same data, same gesture. This is
`adw-fe-15-two-screens-one-language` applied to the new surface: if the two
screens render headroom differently, the ticket has failed even if both work.

## 3. The one real difference from ticket 02

The run screen has its **own** SSE path and its own swapped fragment:

```
/run-events -> document.getElementById('run-console').innerHTML = e.data
```

`renderRunHeader` is inside `#run-console` (`render-run.ts:363`), so it is
destroyed on every push exactly as the board header is. **The same three-way
split applies** — chips inside and re-mounted, panel outside, open state in a
closure outside — but against `#run-console` rather than `#grid-body`.

If `adw-usage-02` left that split as board-specific code, generalise it here
rather than copying it. One implementation, two mount points.

Second difference, smaller: the run screen already owns `.dock`
(`render-run.ts:367`) and a node-click handler that fills it. The usage panel
must not collide with it — `summary.prototype.ts` §2b records what that
collision looks like (two bottom panels at once) and how it was resolved
there. Read it before choosing a placement.

## 4. Red tests first (Art. I)

Prior art: `test/web/render-run.test.ts`, and `test/web/server.test.ts` for
the injected-probe route, both of which `adw-usage-02` will already have
extended.

1. the run screen's rendered header carries the chips
2. the run screen's swap handler re-mounts them after `#run-console` is
   replaced
3. the panel is rendered **outside** `#run-console`
4. opening the usage panel does not open or disturb `.dock`, and a node click
   does not disturb the usage panel
5. a `{ok:false}` reading renders "unavailable" on the run screen too — and
   the run screen still renders

## 5. Out of scope

- Any change to what the panel contains. If ticket 02's payload is wrong,
  that is a bug against 02, not a redesign here.
- Per-run attribution of subscription usage — spec §"Out of Scope". The run
  screen showing account-level headroom next to run-level token figures must
  not imply the two are the same measurement; keep them visually separate.
- A third surface. Two screens is the whole of `adw-fe-15`'s language.

## 6. Verify

- `bun run lint && bunx tsc --noEmit && bun test` all green
- new tests red before implementation, green after
- manually: open a live run, confirm the chips survive ≥10 `/run-events`
  pushes, the panel stays open across them, and clicking a Gantt block still
  opens `.dock` normally with the usage panel unaffected
- the two screens' headroom surfaces are visually identical side by side
