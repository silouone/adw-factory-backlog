---
id: adw-fe-24-the-live-tick-erases-every-prompt-from-the-panel
type: bug
status: done
priority: 1
created: 2026-09-18
caps: {minutes: 120, turns: 600}
depends: []
attempts: [{"runId":"adw-fe-24-the-live-tick-erases-every-prompt-from-the-panel-1789738858464","branch":"adw/adw-fe-24-the-live-tick-erases-every-prompt-from-the-panel","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-24-the-live-tick-erases-every-prompt-from-the-panel-1789738858464/workspace","outcome":"blocked","provider":"claude","model":"sonnet","pr":"https://github.com/silouone/adw-factory/pull/79"}]
---
# The run panel renders every compiled prompt, then the first SSE tick throws them all away one second later

> Observed 2026-09-18 by the operator on the run screen for
> `adw-fe-15-two-screens-one-language`:
> *"the bottom panel in the focus task tab, when looking at the plan or build
> etc, they ARE NOT displaying their prompt in the bottom panel which is the
> entire point of the thing. We got KPI on the steps but we should have the
> PROMPT visible for these steps."*

The operator is right, and the prompt is not missing from disk, from the
journal, or from the renderer. It is rendered correctly by the server, sent to
the browser, and then **deleted by the page's own live-refresh loop** within
roughly one second of load — before a human can read it.

## Evidence

`adw-fe-03-prompt-persistence` did its job. For the run above's sibling
`adw-pr-01-resolve-a-card-to-its-pull-request-1789714981882`:

```
runs/<runId>/prompts/0001-plan.txt          9,665 bytes
runs/<runId>/prompts/0002-build.txt        41,529 bytes
journal: {"type":"prompt","node":"plan", ...,"ok":true}
journal: {"type":"prompt","node":"build",...,"ok":true}
```

`loadRunDetail` resolves both to `PromptView{status:"ok"}`. Replaying two
banked journals through the real `renderRunPanel`, once down the static path
and once down the live path the SSE route actually uses:

| run | state | panel | `prompt-text` | `drawer-kind` | bytes |
|---|---|---|---|---|---|
| `adw-pr-01-…-1789714981882` | running | **static** | 2 | 8 | 70,740 |
| `adw-pr-01-…-1789714981882` | running | **live tick** | **0** | **0** | 15,146 |
| `adw-fe-21-…-1789677026767` | finished (blocked) | **static** | 3 | 13 | 104,034 |
| `adw-fe-21-…-1789677026767` | finished (blocked) | **live tick** | **0** | **0** | 31,956 |

The live fragment loses **every** drawer, not only the prompt: node, kind,
attempt, system prompt and agent config all vanish with it. What survives is
exactly the screenshot the operator sent — `card-hd` + `card-stats` + the
model/tool split bar, and nothing below it.

## Root cause — one argument

`src/web/server.ts:595`, inside `handleRunEvents`'s poll tick:

```ts
renderRunPanel(header, view, new Map(), now, summary)
```

The empty map is deliberate and commented (server.ts:586-592), mirroring
`composeRunView`'s stated "no per-tick prompt-file read per block" stance
(`src/web/run-view.ts:166-172`). `renderCardBody` then takes its
`drawer !== undefined` branch to the empty string, and every step section
renders KPI-only.

**The reason it is invisible to the author and total for the operator** is the
second half: `src/web/render-run.ts:491`.

```js
var es = new EventSource("/run-events" + location.search);
```

That line is **unconditional**. It is not guarded on `header.state`, and
`staleScript` only toggles a CSS class on error/open — it never closes the
stream. So *every* run page opens the SSE stream, finished runs included, and
`handleRunEvents`'s `tick()` fires **immediately on stream start** (it is
called from `ReadableStream.start` before the poll is armed). The very first
push replaces `#dock` wholesale with the drawer-less panel.

The static render this ticket proves correct is therefore **never the render an
operator reads.** It exists for about one second. Every run, every time.

## Why it matters

- **It is the panel's stated purpose.** `specs/adw-v1.7-run-panel.md` makes the
  panel the screen's primary reader — *what did this run do*. A step's compiled
  prompt is the only artefact that answers what an agent was actually asked.
- **It silently voids `adw-fe-03`.** Prompt persistence was built because no
  prompt the factory composes was durable. It is durable now, indexed, hashed —
  and unreadable in the one surface built to read it.
- **It hides the profile work.** `adw-profile-01` makes model and effort vary
  per stage; `DrawerView.config` and `systemPrompt` are how an operator sees
  what a stage resolved to. Both are in the same discarded drawer.
- **It hides the `mismatch` state.** `resolvePrompt` can report a truncated or
  externally edited sidecar. That warning is rendered and then erased too.

## The fix, and why the original author punted

Do **not** naively rebuild drawers per tick. `buildDrawerView` reads and
`hashPrompt`s every sidecar; `build`'s is 41KB, `SSE_POLL_MS` is 1,000ms, and
that re-read is precisely the cost the empty map was protecting.

The sound fix is a **per-stream memo keyed on the sidecar path**.
`makePromptSink` assigns a globally unique zero-padded sequence per call, so a
sidecar is **write-once and never overwritten** — one read plus one
`hashPrompt` verify per path holds for the life of the stream. The memo lives
in `handleRunEvents`'s own closure beside `state`, is bounded by the number of
agent nodes in one run, and dies with the connection.

## Requirements

- [ ] An SSE tick's dock fragment carries the same drawers the static page
      does: for an agent block whose journal holds an `ok:true` `prompt`
      record, the fragment contains that prompt's text.
- [ ] `systemPrompt`, `config`, `kind` and `attempt` survive the live tick too
      — the whole `DrawerView`, not the prompt alone.
- [ ] The `mismatch` and `not-persisted` states survive the live tick and still
      state their reason, rather than degrading to an empty section.
- [ ] Each prompt sidecar is read and hashed **at most once per SSE
      connection**, not once per tick. Justify the memo's soundness from
      `makePromptSink`'s write-once sequence in a comment, per Art. IX.
- [ ] The prompt block is **readable**, not merely present: `.prompt-text`,
      `.prompt-not-persisted` and `.prompt-mismatch` have **no CSS rule at all**
      today (only the `.drawer-prompt` wrapper does, `render-run.ts:1033`). A
      41KB `<pre>` with no `white-space`/`max-height`/`overflow` inside
      `body:has(#dock){overflow:hidden}` does not read as a prompt. It must
      wrap, scroll within its own section, and never move the panel's outer
      layout or the Gantt above it.
- [ ] `composeRunView` stays drawer-free. Its contract is correct; the memo
      belongs at the SSE edge that owns a connection's lifetime, not in the
      pure composition every caller shares.
- [ ] Do not regress the `body !== lastSent` frame suppression. Prompt bodies
      make a changed frame ~50KB larger; a live run changes most ticks. Measure
      it, state the number, and only split the payload if the measurement
      demands it.

## Verify

- [ ] Red test first, in `test/web/server.test.ts`: the slice of a
      `/run-events` push **after `DOCK_SPLIT_MARKER`** contains `prompt-text`
      for an agent block whose journal carries an `ok:true` `prompt` record.
      RED today at `src/web/server.ts:595` — the count is 0.
- [ ] Red test: the same fragment carries `drawer-kind` / system-prompt /
      config for that block. RED today.
- [ ] Red test: a second tick on the same connection performs **no additional
      read** of an already-read sidecar — assert against an injected counting
      `ReadPromptFile`, not a timing.
- [ ] A `prompt` record with `ok:false`, and a sidecar whose bytes/hash
      disagree with the journal, each render their own live-tick state with
      its reason — neither throws, neither blanks.
- [ ] Regression: a deterministic or gate block still renders no prompt section
      at all on the live tick (it has no `DrawerView.prompt` by construction).
- [ ] Regression: `test/web/run-view.test.ts`'s "sans drawers" property for
      `composeRunView` still holds unchanged.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] Open `/run?id=<a banked run with prompt sidecars>` in a browser, wait
      past the first tick, select `plan` and then `build` in the rail, and read
      both prompts end to end without the panel resizing the Gantt.

## Out of scope

- **Client-side DOM preservation.** `render-run.ts:374-378` states the panel is
  re-rendered server-side every tick and the client re-applies its own
  `active`/`mdOn` state against fresh markup, never patching old DOM in place.
  A "preserve the drawer across swaps" client fix contradicts that design; fix
  the payload, not the swap.
- **Guarding `EventSource` on run state.** A finished run's page opening a poll
  loop is its own inefficiency and is named here only as the reason this defect
  reaches completed runs too. It is not this ticket's fix — once the payload is
  whole, a finished run's stream is harmless.
- Changing what any prompt contains, `assemble-prompt`, or `prompt-sink`'s
  storage format. The data is correct; only its delivery to the panel is not.
- The Gantt axis (`adw-fe-22`) and timeline zoom (`adw-fe-23`).
