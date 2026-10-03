# silou-hq — v2 spec: the action ring

> **Status:** approved by the operator 2026-10-03.
> **Supersedes:** parts of `docs/spec-v1.md`, through the amendment below. Everything in v1 that
> this spec does not amend still binds.
> **Labels:** `ready-for-agent`.
> **Sources:**
> - the accepted interaction prototype: branch `proto/action-ring` @ `5eecf24`, local only, never merged;
> - its decision record, `PROTOTYPE-NOTES.md` on that branch;
> - the grilling of 2026-10-02 (Q1–Q5);
> - the operator round of 2026-10-03, which settled lens, variant, ASK HQ scope, lease, undo, seams, the always-on arc and the zoom clamp.
>
> The prototype is the reference for feel. Where this spec and the prototype disagree, this spec wins.
> The disagreements are deliberate and are listed in Further Notes.

## Amendment to v1 (lands with this spec, before any v2 ticket is dispatched)

v1's rule #1 ("v1 is read-only. No write route, no action button. Every non-GET returns 405.
Writes are v2, behind a lease and an action ledger.") is replaced.

**It lands in the same commit that approves this spec,** because v1's "no action button" binds
every bezel ticket, not only the write path. That commit:
- edits `CLAUDE.md` (= `AGENTS.md`);
- adds a one-line pointer here to `docs/spec-v1.md`, which otherwise keeps its text.

The replacement wording:

> 1. **Writes are local, allowlisted and ledgered.** Exactly one write route exists: `POST /action`.
>    Every other non-GET returns 405. An action names a node id the graph recorded and a verb from
>    that node kind's fixed list. The client never sends a path, a shell string or a command.
>    Control verbs act on **this Mac** only. Every attempted action, refused ones included, is
>    appended to the action ledger in `cache/`. Destructive verbs need an inline confirmation in the
>    UI. **There is no lease.**

Consequences for the other non-negotiables:
- **Rule #2 (HQ owns no data)** still holds. The ledger and the previous-version store live in
  gitignored `cache/` and are HQ's only new state.
- **Rule #4 (reads are allowlisted)** is extended to writes. A write target is always resolved on
  the server from a node id, never taken from the request. See *The write surface* for the two
  cases where the target is not yet in the graph.
- In v1's Out of Scope, "every write action… all of this is v2, behind a lease and an action ledger"
  becomes: "triggering routines (this Mac) is v2, with a ledger and no lease. The other v1 write
  exclusions still stand."

## Problem Statement

v1 lets me see my whole setup, but not act on it.
- When the rings show a routine failing every day (brain-hub's `d1-channels`, `d2-redshift` and
  `d3-snapshot` still fail because their `run-*.sh` scripts are gone), HQ can tell me. To fix it I
  then open a terminal, find the plist, read the log, recreate the script and `launchctl` it back.
- Showing a skill in Finder or opening a repo in VS Code means leaving HQ and finding the path by hand.

Interaction on the rings also has rough edges:
- hovering only shows a tooltip;
- there is no way to see what a node can do;
- plugins and MCP servers crowd the runtimes ring;
- CLAUDE.md squats the centre;
- the routines ring doesn't read as a clock;
- a Mac trackpad pinch and two-finger pan behave like a mouse wheel.

The centre of the rings, the most valuable spot on the page, does nothing.

## Solution

v2 turns the rings into an **instrument**: point at anything and it tells you what it can do.

- **The hover model.** A node catches the pointer (**magnet**). Resting on it swells its
  neighbourhood (**swell**). Resting longer pops a preview of its action dial in place (**ghost**).
  Moving onto the ghost makes it live (**promote**). Click and hover give the same UX.
- **The action bezel.** A circular dial around the focused node, with four fixed sectors:
  - **READ** (north);
  - **OPEN** (east): Finder, VS Code;
  - **CONTROL** (south): this Mac's routines;
  - **LINKS** (west): the node's neighbours, paged.

  The dial sits on an ember focus zone that darkens only around it.
- **The routines console.** A deep view: the rings shrink into a corner mini-ring, and one routine
  fills the page with its verdict, timeline, logs and an editor for its schedule and script. You can
  also create a new routine there. Every change is confirmed, ledgered and undoable.
- **The landing, refined.**
  - The inner ring is LLM runtimes only. Plugins and MCP servers get their own track. CLAUDE.md
    docks beside its runtime.
  - The hooks and live-sessions rings become visible.
  - The routines ring becomes a true 24 h clock, with always-on routines on their own arc.
  - Any ring can be focused.
  - The header tucks away when you dive in.
  - Mac trackpad gestures work natively.
- **ASK HQ at the centre.** A speaker cone that opens a modal conversation, with a voice spectrum
  (Mirror) that thinks, speaks and listens. v2 ships the shell and a runtime-agnostic agent seam.
  No agent backend is connected yet, and the shell says so honestly.

## User Stories

**Landing: the rings**
1. As the operator, I want the inner ring to hold only LLM runtimes (claude code, codex, ollama), titled `RUNTIMES · ANY LLM`, so that "any LLM is a tool" reads at a glance.
2. As the operator, I want plugins and MCP servers on their own track between the runtimes and the skills (plugins as squares, MCP servers as diamonds, muted teal), so that they stop crowding the runtimes.
3. As the operator, I want a plugin whose label equals a runtime's label shown as `<label> · plugin`, so that I never see two identical "codex" nodes.
4. As the operator, I want each runtime's instruction file (CLAUDE.md, AGENTS.md) docked beside its runtime, so that the centre is free and I see which runtime reads which instructions.
5. As the operator, I want a runtime whose instruction file is missing (today: codex has no `~/.codex/AGENTS.md`) marked as missing, so that I notice the gap.
6. As the operator, I want the hooks and live-sessions rings drawn as visible, titled bands with counts (`HOOKS · ON EVENT · n`, `LIVE SESSIONS · n · k WAITING`), so that I can find them without hunting.
7. As the operator, I want the routines ring to read as a 24-hour clock, with these parts, so that I read it like a watch:
   - 00 at the top and numerals every 3 h;
   - quarter-hour ticks;
   - a today-so-far arc;
   - a labelled ember `NOW HH:MM` hand.
8. As the operator, I want always-on routines on their own short arc beside the clock, so that they don't bunch at the bottom of the dial and their labels never overlap.

**Landing: navigation**
9. As the operator, I want to click a ring's line or band (not a node) to focus that ring, so that I can study one ring alone. The rest of the page dims behind a veil, and every item on the ring is named.
10. As the operator, I want to release a ring focus by clicking the ring again, pressing Esc, or selecting anything, so that leaving is obvious.
11. As the operator, I want the header to tuck away whenever I zoom in or open a bezel, the console or ASK HQ, leaving search as a small tab, so that the rings get the whole screen while I work.
12. As the operator on a Mac trackpad, I want pinch to zoom at the cursor and two-finger move to pan 1:1 with native momentum, so that HQ feels like a native Mac surface.
13. As the operator with a mouse, I want the wheel to keep zooming, so that a mouse still works.
14. As the operator, I want panning bounded so that the rings' centre stays on screen, so that I can never lose the rings.
15. As the operator, I want a focused node centred exactly, without a pinch snapping the view back afterwards, so that focusing is stable.

**Hover model**
16. As the operator, I want a node to catch my pointer and lean slightly toward it, and to stay caught for a moment after I slip off, so that small targets are easy to hit.
17. As the operator, I want a near-miss click during that grace period to select the caught node, so that I don't need pixel precision.
18. As the operator, I want resting on a node to slowly swell its neighbourhood and name its neighbours, so that I see what it relates to without clicking.
19. As the operator, I want resting still on a node for about ¾ s to pop its action dial in place as a preview, so that I can see what it can do without committing.
20. As the operator, I want simply passing over nodes never to trigger a preview, so that moving across the rings stays calm.
21. As the operator, I want moving from the preview onto one of its sectors to make it the live dial in place, with no camera move, so that hover and click lead to the same place.
22. As the operator, I want a hover-opened dial to close shortly after I leave its zone, and a click-opened one to stay until Esc or a click elsewhere, so that each behaves as I'd expect.
23. As someone who prefers reduced motion, I want no lean, no swell wave and no bloom animation, so that the page stays calm. The same states still appear.

**The action bezel**
24. As the operator, I want every node's actions in a dial with four fixed sectors (READ north, OPEN east, CONTROL south, LINKS west), so that I always know where to look.
25. As the operator, I want an empty sector still drawn (dashed), so that the layout never reshuffles from one node to the next.
26. As the operator, I want READ and CONTROL actions engraved on curved tracks with upright text, so that I can read them without tilting my head.
27. As the operator, I want OPEN actions and links as radial spokes with readable labels on both sides, so that the dial stays legible.
28. As the operator, I want a node's neighbours paged in LINKS (14 per page) and turned by wheel, swipe or a `‹ n / N ›` control in the readout, so that a busy node stays readable.
29. As the operator, I want one swipe or wheel notch to turn exactly one page, so that paging is deliberate.
30. As the operator, I want a readout under the dial with the node's kind, label and one fact, which switches to the hovered action's plain-language hint, so that I know what an action does before I click it.
31. As the operator, I want selecting any node (canvas, link spoke or widget row) to glide it to the exact screen centre and bloom its dial there, so that my eyes stay at the centre and the interface adapts.
32. As the operator, I want the dial's hub, glow and ticks in the selected node's ring hue, and each sector in its own hue (READ violet, OPEN teal-blue, CONTROL brass, LINKS brand blue), so that the dial feels part of the app.
33. As the operator, I want the dial to sit on an ember focus zone that darkens only around it, never page-wide, with no border, and that grows and shrinks smoothly with the dial's reach, so that focus is local and calm.
34. As the operator, I want "Show in Finder" and "Open in VS Code" for every file-backed node, so that I can jump to the real file in one click. VS Code opens the enclosing git root, or the folder for a memory file.
35. As the operator, I want READ to offer the routine console, the last output, the full-screen file and the project page wherever they apply, so that reading is one click from any node.
36. As the operator, I want each action's toast to name its verb in the past tense ("Started", "Paused", "Schedule saved"), and a failed action to say why, so that I trust what happened.

**Routines: control (this Mac)**
37. As the operator, I want to run a routine now, outside its schedule, so that I can test a fix immediately.
38. As the operator, I want to pause a routine (off its schedule, surviving a reboot) and resume it, so that I can silence a noisy routine without deleting it.
39. As the operator, I want to stop a run in progress, with an inline confirmation, offered only while it runs, so that I can kill a stuck run safely.
40. As the operator, I want to remove a routine with its own inline confirmation, the plist going to the Trash and a copy kept for restore, so that removal is safe and reversible.
41. As the operator, I want M1 routines shown read-only, with CONTROL saying "Read-only on the M1", so that I never think I changed something I didn't.

**Routines: the console (deep view)**
42. As the operator, I want opening a routine's console to shrink the rings into a clickable corner mini-ring, so that I keep my bearings and can return in one click or with Esc.
43. As the operator, I want a side column of this Mac's routines with a health dot each and an "N need you" count, so that I can triage from one list. Broken and failing are red, running is green, ok is brass; M1 routines are greyed.
44. As the operator, I want each routine introduced in one sentence ("Runs every day at 07:00 · next tomorrow at 07:00 · in 8 h 54 min"), so that I know its rhythm without reading a plist.
45. As the operator, I want a 24 h timeline with one lane per routine, its scheduled ticks and a now line, so that I see the day's rhythm across routines.
46. As the operator, I want a "last run · exit · what happened" verdict that names a missing script when the routine is broken, with a **Recreate the script** call to action, so that the brain-hub failure is one click from fixed.
47. As the operator, I want stderr, stdout and plist tabs, with repeated identical log lines folded into one line with a ×N badge, so that 120 copies of the same error read as one.

**Routines: the editor**
48. As the operator, I want to choose when a routine runs (Every day, Some days, Every N min, Always on) with a 24 h dial knob snapping to 5 minutes, arrow keys ±5 and weekday pills, so that I set a schedule without writing XML.
49. As the operator, I want the schedule echoed as a sentence plus the next run, so that I know exactly what I set.
50. As the operator, I want to edit the script in a code editor with a gutter, bash highlighting and Tab inserting two spaces, so that small fixes don't need another app.
51. As the operator, I want a live diff of my script edit (+/− counts), so that I see exactly what I'm about to save.
52. As the operator, I want a save bar that summarises what changed ("6 lines changed · schedule changed"), with Discard, Save (⌘S) and an inline "Save and reload" confirmation, so that nothing is written by accident.
53. As the operator, I want a missing script pre-filled with a template (shebang, `set -euo pipefail`, `cd` into the working directory), so that recreating it starts from a sane base.
54. As the operator, I want to restore a routine's previous plist and script after an edit or a removal, so that any change HQ made can be undone.

**Routines: create**
55. As the operator, I want to create a new routine from the same editor (name, the repo it runs in, schedule, script), so that I never hand-write a LaunchAgent again.
56. As the operator, I want a new routine's name constrained to `com.silouane.<name>`, its script placed under the chosen repo's `routines/` folder, and its logs always set to `~/Library/Logs/silou-hq/<name>.{out,err}`, so that every new routine has a known shape and a visible last run.
57. As the operator, I want creating a routine that already exists, or a script that already exists, to be refused, so that create never overwrites.

**The ledger and safety**
58. As the operator, I want every attempted action recorded with its time, node, verb, the exact command run and its result, so that I can always answer "what did HQ change on my Mac?"
59. As the operator, I want refused actions recorded too, with the reason, so that probing or bugs are visible.
60. As the operator, I want HQ to refuse any action whose node id the graph doesn't know, whose verb isn't in that kind's list, or that targets the M1, so that the write surface is exactly what the UI offers.
61. As the operator, I want HQ to refuse a write request that doesn't come from HQ's own page, so that another site open in my browser can never drive my LaunchAgents.
62. As the operator, I want the server to resolve every write target from the node id, never from the request, so that HQ cannot be used to write an arbitrary file.

**ASK HQ (the shell)**
63. As the operator, I want an ember speaker cone labelled `ASK HQ` at the centre of the rings, so that the most valuable spot on the page is the way to ask.
64. As the operator, I want clicking the cone to open a modal conversation without moving the camera, so that the rings stay where I left them.
65. As the operator, I want everything outside the modal unreachable while it's open (no hover, preview, dial, pan or zoom), and leaving to take ✕ LEAVE or Esc, so that nothing opens by accident.
66. As the operator, I want only a local halo dimmed around the centre, never the whole page, so that it feels like the rest of HQ.
67. As the operator, I want a composer with attach (also drop files onto the centre, shown as chips with size and ×), a text field, a mic and a spoken-reply toggle, so that the shell is complete when a backend arrives.
68. As the operator, I want the reply rendered inside the cone as one formatted message (headings, lists, code, bold), so that the answer lives where I asked.
69. As the operator, I want the cone's spectrum to show thinking (organic noise clusters), speaking (out, in ember) and listening (in, in gold, following my mic level), so that HQ's state is visible without text.
70. As the operator, I want the spectrum to read like a voice (irregular, asymmetric, transient spikes), never a symmetric equaliser, so that it feels alive.
71. As the operator, I want ASK HQ to say plainly that no agent is connected yet, rather than fake an answer, so that missing sources are never faked (v1, story 41).
72. As the factory's builder, I want the agent behind ASK HQ behind one runtime-agnostic adapter, so that the backend spec can plug in claude, codex or ollama without touching the shell.

**Quality**
73. As the operator, I want every new view to have an honest empty and offline state (no routines, launchd unreadable, graph still loading), so that HQ never shows a broken page.
74. As the operator, I want the first click after a page load never lost while the graph loads, so that HQ responds from the first moment.
75. As the operator on a smaller display, I want the dial, focus zone and cone to scale down with the viewport, so that the instrument still fits on the laptop.

## Implementation Decisions

### Modules

New or changed areas:

- **The write surface** (server): the action route, verb allowlist, target resolution, executor and ledger.
- **The routine reader** (server): this Mac's routine detail, extending the graph build's routine data from hq-10 and hq-12 rather than forking it.
- **The schedule codec** (pure): editor schedule ↔ plist keys ↔ sentence ↔ next run.
- **The bezel** (web): geometry, sectors, paging, transit and the focus zone.
- **The hover model** (web): magnet, swell, ghost and promote, as a pure state machine plus a canvas renderer.
- **The action catalogue** (web): kind → sectors and actions.
- **The console and editor** (web, Preact): the side column, overview, logs, schedule dial, code editor, diff and save bar.
- **ASK HQ** (web): the cone, modal, composer, spectrum, and the agent adapter seam.
- **Ring changes** (web): pure layout additions in the ring-layout module, and drawing in the canvas entry.

### The write surface

- **Route:** `POST /action` with a JSON body `{ nodeId, verb, payload? }`. It is the only non-GET route that doesn't get a 405.
- **Origin guard.** The request is refused (403) unless all four hold:
  - `content-type` is `application/json`;
  - `Host` is exactly `127.0.0.1:<port>` or `localhost:<port>`, with the port taken from config;
  - `Origin` is exactly `http://127.0.0.1:<port>` or `http://localhost:<port>`;
  - `Sec-Fetch-Site`, when present, is `same-origin`.

  The allowed set comes from config and is **never derived from the request's own `Host`**: a
  derived origin would let a DNS-rebinding page match itself.

  This blocks cross-site form or fetch posts to `127.0.0.1` from any other tab, the embedded adw
  web frame included.
- **The verb allowlist,** pure and exhaustive:

  | Node kind | Verbs |
  |---|---|
  | any file-backed node (memory, report, spec, skill, command, subagent, hook, core) | `reveal`, `code` |
  | `repo` | `reveal`, `code` |
  | `routine` on this Mac | `run`, `pause`, `resume`, `cancel`, `schedule`, `script`, `remove`, `restore`, `reveal`, `code` |
  | `routine` on the M1 | none: refused with "read-only on the M1" |
  | HQ's own routine (`com.silou.hq`) | `reveal`, `code` only: CONTROL shows a disabled "This is HQ" |
  | the pseudo-node `routines:new` | `create` |
  | a removed routine `routines:removed/<label>` | `restore` |

  - A removed routine's pseudo-id is derived by the server from `cache/previous/`, not from the
    graph. Removing a routine drops its node at the next rebuild, and this keeps its undo reachable.
    Removed routines that still have a snapshot are listed in the console's side column.

  - **Destructive** (confirmed inline in the UI): `cancel`, `script`, `remove`, `restore`.
  - **Confirmed in the save bar:** `schedule` and `create`.
- **Refusals** answer JSON `{ ok: false, error }` and are ledgered:
  - 404: unknown node id;
  - 403: verb not allowed for the kind, M1 target, origin guard;
  - 409: `create` over an existing label or script; `restore` with nothing to restore;
  - 400: a payload fails validation.

  Errors name the node id and the verb.
- **Target resolution,** always on the server. The client never sends a path.
  - `reveal` and `code` resolve the node's file from **this Mac's** entries in the graph's recorded
    file allowlist (the one `/file?id=` uses). M1 entries are refused.
  - `repo` nodes resolve through a separate this-Mac repo resolver, fed by the graph build's local
    repo scan.
  - `code` opens the enclosing git root, or the containing folder for a memory file.
  - Routine verbs resolve the plist from the routine node, and the script from the plist's
    `ProgramArguments`: the first `.sh/.py/.ts/.js/.mjs/.rb` argument, made absolute against
    `WorkingDirectory`; or `-m <module>` resolved under the working directory or its `src/`.
  - **There is no fallback.** A routine with no matching argument has no editable script, and
    `script` is refused with 403. The prototype fell back to `ProgramArguments[0]`, which is the
    interpreter (bun, python).
  - A resolved script path must be under `$HOME`, outside `~/Library`, and not equal to
    `ProgramArguments[0]`.
  - **`script` takes `{ content }` only. The prototype's client-sent path is removed.**
- **`create` validation:**
  - `label` must match `^com\.silouane\.[a-z0-9][a-z0-9-]{0,47}$`;
  - `runsIn` must be a `repo` node id present on this Mac;
  - the script is written to `<repo root>/routines/<name>.sh`, which follows brain-hub's existing
    convention and must not exist;
  - the plist goes to `~/Library/LaunchAgents/<label>.plist`, which must not exist;
  - logs are always `~/Library/Logs/silou-hq/<name>.{out,err}`.
- **Script content** is at most 40 000 characters (the prototype's read cap), UTF-8, and gets no
  other interpretation. It is written as a file, never executed by `/action` itself except through
  `run`.
- **Commands per verb** (`gui/<uid>/<label>`, uid from the process). They come from the prototype's
  `wouldRun`, which v2 executes for real:

  | Verb | Command |
  |---|---|
  | `run` | `launchctl kickstart` |
  | `cancel` | `launchctl kill SIGTERM` |
  | `pause` | `launchctl disable` then `bootout` |
  | `resume` | `launchctl enable` then `bootstrap gui/<uid> <plist>` |
  | `schedule` | snapshot, rewrite the plist's schedule keys, `bootout`, `bootstrap` |
  | `script` | snapshot, write the script (keeping its mode, `+x` when new) |
  | `remove` | snapshot, `bootout`, move the plist to `~/.Trash/` |
  | `restore` | write back the newest snapshot, `bootstrap` if the plist came back |
  | `create` | write the script and plist, `bootstrap` |
  | `reveal` | `open -R <file>` |
  | `code` | `open -a "Visual Studio Code" <dir>` |

- **The executor** is injected. It receives an argv array (never a shell string) or a typed
  file-write instruction, and returns `{ exitCode, stderrTail }`. Production spawns with a timeout;
  tests record. `reveal` and `code` are fire-and-forget.
- **The ledger:** `cache/actions.jsonl`, append-only, one line per attempt. It holds:
  - `at`, `nodeId`, `verb`;
  - `outcome ∈ done | refused | failed`;
  - `argv[]` (or the file-write summary);
  - `exitCode`, `error` or `stderrTail`;
  - for script writes, the content's length and a sha256, never the content itself.

  `GET /ledger.json` returns the newest 200 entries.
- **Previous versions** (undo): before every `schedule`, `script` and `remove`, the current plist
  and script are copied to `cache/previous/<label>/<epochMs>.{plist,script}`. `restore` writes back
  the newest pair and consumes it. Up to 10 are kept per label.
- **Concurrency:** actions are serialised per label. A second write on a label while one is in
  flight gets 409.

### The routine reader

- `GET /routine?id=<routine node id>` returns the console's detail for a routine the graph knows.
  - **This Mac:** plist (parsed and raw XML), resolved script (path, exists, content up to 40 000
    chars), working directory, schedule, `launchctl` pid, last exit and disabled state, and log
    tails (120 lines each, through the existing bounded-tail rules).
  - **M1:** what the snapshot holds.
- **Health** is `unloaded | broken | running | failing | ok`, with a `why`. It extends hq-12's
  routine health (failed = no PID and a non-zero last exit) with `broken` (script missing) and
  `unloaded`. It never redefines "failed".
- Owned labels only: the same `com.silou*|com.user.*|io.sabado*` filter as the graph build.
- `launchctl` is read through injected readers with a timeout. A routine whose state can't be read
  is `unloaded` with that `why`, never an error page.

### The schedule codec (pure)

```ts
// from the prototype's editor model, trimmed to the decision
type Schedule =
  | { kind: "daily"; at: HHMM }                        // StartCalendarInterval {Hour, Minute}
  | { kind: "days"; at: HHMM; weekdays: Weekday[] }    // one calendar dict per weekday
  | { kind: "every"; minutes: number }                 // StartInterval = minutes·60
  | { kind: "always" };                                // KeepAlive true
```

- Functions: `toPlistKeys(schedule)`, `fromPlist(plist)`, `sentence(schedule)`,
  `nextRun(schedule, now)`.
- They **reuse hq-10's launchd semantics** (calendar, interval, weekday, always on); one definition
  only.
- A plist this model can't express (several times a day, month or day keys) opens the editor
  read-only for the schedule, with "edit the plist to change this".
- Times snap to 5 minutes.

### Landing changes (from PROTOTYPE-NOTES §2)

**Rings:**
- **Runtimes:** band ±0.016 U, line 1.4 px at .75. Title at R.rt+0.022.
- **Plugins and MCP:** `R_EXT = 0.141` U, colour `#7FA7B5`, plugins as squares and MCP servers as
  diamonds, size 4.6. Title `PLUGINS · MCP · n` at angle 0.8π.
- **Instruction files:** docked at runtime angle +0.3 rad, radius R.rt+0.03, label above.
- **Hooks:** band ±0.008, brass .5, dashed `[3,4]`.
- **Sessions:** band ±0.008, live blue .42, dashed `[2,4]`.
- **Ring titles:** at angle −π/2−0.8.

**The routines clock:**
- 96 ticks: 12 / 8 / 5 / 2 px at 6 h / 3 h / 1 h / 15 min.
- Numerals every 3 h, 00 bolder.
- A today-so-far arc, 7 px at .16.
- An ember now hand ending in a dot, labelled `NOW HH:MM`.

**The always-on arc:** always-on routines sit on a short arc just outside the clock, centred at
the bottom (6 o'clock) and spanning at most 40°, evenly spaced, named when the routines ring is
focused. They are no longer placed by time.

**The `band()` drawing fix** carries over: `moveTo` between the outer and inner arcs.

**Ring focus:**
- Focusable rings: runtimes, plugins·mcp, skills, memory, live sessions, hooks, routines, projects.
- The veil is **one radial gradient**: darkness .6, clear over [r0, r1], with a fade of .03 U on
  thin rings and .05 U on wide ones. It is drawn last.
- The focused band gets a tint of .05 (thin) or .025 (wide) and **no border**.
- Fades take 320 ms, ease-out cubic.
- Items are named when there are 60 or fewer, or when k > 1.8.
- Title: `TITLE · n — click the ring again or press esc`.

**Header tuck:**
- It tucks at k > 1.15, or while a bezel, the console or ASK HQ is open. It untucks below 1.08.
- 0.45 s, `cubic-bezier(.2,.9,.2,1)`. The brand rises 14 px and fades.
- Search collapses to a ⌕ tab, and its label returns on hover or focus.

**Trackpad:**
- Pinch is `ctrlKey` + wheel, zooming at the cursor by `exp(−clamp(deltaY, ±50)·0.012)`.
- A two-finger move pans 1:1 with no easing.
- A mouse wheel (`deltaMode ≠ 0`, or `deltaX = 0` with `wheelDeltaY % 120 = 0`) still zooms.
- Pan keeps the rings' centre within 8–92 % of the screen.

**The zoom clamp:** while a node is focused, `clampView` centres its bound on the focused node
instead of on the rings' centre, so exact focus centring and a later pinch agree. The 8–92 % pan
bound still applies. This is pure, and it lives in the ring-layout module.

**First click:** pointer input is queued until the first graph load completes, then replayed once.

### The hover model (PROTOTYPE-NOTES §3)

A pure state machine of `(state, event, now) → state`, with events pointer-move, pointer-leave,
click, tick and key. The renderer only draws its state.

```ts
// the prototype's constants; they are decisions, not tuning knobs
MAGNET = { graceMs: 380, releasePx: 64, lean: 0.24, maxLean: 7, spring: 0.28 } // ring snap-in 140 ms from +9 px
SWELL  = { t0: 300, t1: 950, sigmaPx: 120, wave: "sin(t/480 − d/34)", hoveredScale: 1.3, named: 10 }
GHOST  = { stillMs: 750, maxSpeedPxPerMs: 0.3, popInMs: 180, popFromScale: 0.9, fadeOutMs: 165 }
PROMOTE = { holdRadius: "HOT + 22", brightenMs: 220, hoverCloseMs: 300 }
```

- The contact clock resets on a node change, when speed exceeds 0.3 px/ms, and during the
  magnet's hold.
- A near-miss click during the hold selects the held node.
- No swell inside an open focus zone.
- The ghost is animated by its inner element only. **Never animate the positioned wrapper.**
- The ghost gets the full focus shadow (.95), with dial lines at .7.
- Reduced motion: no lean, no wave, no bloom. The states and timings are otherwise unchanged.

### The action bezel (PROTOTYPE-NOTES §4)

**Geometry.** Screen-space CSS px at the reference display, times the UI scale `s`:
- radii `R0 44, R1 60, R2 122, HOT 138, SPOKE0 162`;
- sectors at fixed positions, ±43° each;
- READ and CONTROL as orbit tracks at `HOT + 30 + i·27`, with text on the curve, the lower half
  reversed so it stays upright, and end dots;
- OPEN and LINKS as radial spokes, with labels flipped on the left half;
- LINKS at 8° apart, 14 per page.

**Paging:**
- Scroll deltas accumulate to `TURN_AT 110`, then a `REST_MS 320` cool-down.
- The dominant axis wins, and a swipe right goes to the next page.
- Swipes are caught on `window` within the dial's reach.
- A page turn rotates the spoke group 40°.
- `‹ n / N ›` lives inside the readout pill.

**Readout:** at reach+10. It shows the kind with a hue inset bar, the label and one fact; on an
action hover, the action's glyph, label and hint.

**Transit:**
- Any selection retracts the bezel, and the camera glides the node to the exact centre.
- Bloom on settle, or after 650 ms at most: scale .55→1 and rotation 32°→0 over 460 ms, sectors
  staggered by 45 ms.
- Retract: scale .45, −40°, 200 ms.

**Focus zone: the ember material** (decided 2026-10-03; smoked glass is dropped):
- base radial `rgba(20,11,5,.94) → rgba(22,12,6,.9) at 66% → rgba(40,20,8,.82)`;
- ember guilloché `rgba(224,135,60,.085)`, 1 px every 5 px;
- fractal-noise grain, soft-light;
- an ember glow band `rgba(224,135,60,.16)` at 74 % fading to `.06` at 88 %;
- the readout pill on `rgba(20,11,5,.95)` with an ember border at .35.

The shape is a JS mask: clear over the hub (R0−4), opaque under the dial, and a 3-stop fade.
- Size: `R = BASE + (reach − BASE)·FOLLOW + PAD`, with `BASE = HOT+14`, `FOLLOW .45`, `PAD 170`.
- The size eases by .075 per frame.
- No page-wide dim, no border, no blur.
- The `?lens` switch and the L key are removed.

**Finish:**
- The hub takes the selected node's ring hue, as a glow, a turning dotted inner ring and 4 notches.
- Sectors are anodized: a radial gradient from R1 outward, an inner rim line, and hue ticks every
  3°, long every 15°.
- A diamond index marker sits at each sector's centre; it fills when the sector is hot, and the hot
  sector gets a drop-shadow in its hue.
- Sector hues: READ `#8f82c4`, OPEN `#4b92b0`, CONTROL `#c3a15a`, LINKS `#7cc5ff`.
- Text `#eef1f6` with a 3.5 px dark halo. Destructive actions in `#ffb4ae`.

**The action catalogue (pure).** `sectors(node, routineDetail?, neighbours)` returns four
sectors in fixed order:
- **READ:** Routine console (this Mac's routines), Last output (`hasOutput`), Read the file
  (`hasFile`), Project page (`cluster`).
- **OPEN:** Show in Finder and Open in VS Code, for nodes that resolve to a local path.
- **CONTROL:**
  - for this Mac's routines: Run now, Pause or Resume (by disabled state), Stop this run (off unless
    running), Edit schedule, Edit script ("missing — recreate" when broken), Remove, and Restore
    previous (only when a snapshot exists);
  - for M1 routines: one disabled "Read-only on the M1";
  - for HQ's own routine: one disabled "This is HQ".
- **LINKS:** neighbours sorted by kind then label, with labels cut to 26 characters.

Toast words: Started, Paused, Resumed, Stopped, Schedule saved, Script saved, Removed, Restored,
Created, Shown in Finder, Opened in VS Code. The "prototype · recorded, not run" label is gone,
because v2 runs for real.

**The UI scale:** `s = clamp(viewportWidth / 3000, 0.7, 1)`. It applies to every screen-space size
in the bezel, focus zone, readout and ASK HQ (cone, reply area, halo). Ring-space sizes stay in U.

### The routines console and editor (PROTOTYPE-NOTES §5)

- **Built in Preact + signals** (v1's widget stack), mounted as a deep view.
- **Opening:**
  - the rings are fitted, then transformed into a mini-ring at centre `(158, h−158)·s`, radius
    `118·s`, in 0.6 s;
  - widgets, zoom controls, the tip and the bezel fade out;
  - clicking the mini-ring or pressing Esc returns and re-selects the routine.
- **Side column:** the count with "N need you" (health broken, failing or unloaded); rows with a
  health dot; **+ New routine**; M1 routines greyed; "◎ back to the rings".
- **Overview:**
  - the sentence from the schedule codec;
  - the toolbar: Run now, Pause/Resume, Stop this run (disabled when idle), ✎ Edit, Finder, VS Code;
  - the 24 h timeline, one lane per routine, with ticks and a now line;
  - the verdict, plus **Recreate the script** when broken;
  - log tabs for stderr, stdout and plist, with the ×N fold.
- **Editor:**
  - "When it runs": a segmented control, the 24 h dial knob (5-minute snap, arrows ±5), weekday
    pills, the sentence and the next run.
  - "What it runs": a gutter, the bash highlighter (comments, strings, `$vars`, keywords), Tab =
    2 spaces, and the live LCS diff with +/− counts.
  - A missing script is pre-filled with the template.
- **Save bar:** the change summary, Discard, Save ⌘S, then an inline "Save and reload". Remove and
  Restore have their own inline confirmations.
- **New routine:** the same editor, plus Name (`com.silouane.` prefix fixed) and Runs in (a picker
  over repo nodes).
- **After any action** the console re-reads `/routine?id=`. The ledger result drives the toast.

### ASK HQ (PROTOTYPE-NOTES §7)

**Idle cone:**
- 64·s px: ember conic surround, a dark cone with ridges every 5 px and directional shading, a
  domed dust cap (inset 34 %), and the label `ASK HQ`;
- on hover, scale 1.08 and an ember drop-shadow.

**Open:**
- A modal. The camera never moves.
- The cone grows to 360·s px and becomes the reply frame, and the dust cap recedes.
- An invisible shield takes every other pointer.
- ✕ LEAVE (above the cone) or Esc leaves.
- The halo is a radial dim of 920·s px, from .78 to 0 at 460·s px.
- The header tucks.

**Composer:**
- 📎 attach, also by dropping files on the centre, shown as chips with size and ×;
- text;
- 🎙 mic: `getUserMedia` level and FFT only, which drive the listening visual locally;
- 🔊 spoken reply via local `speechSynthesis`;
- send.

**Reply:** one formatted message (h4, lists, code, bold) in a round area 280·s px wide inside the
cone.

**The agent seam:**

```ts
type AgentState = "idle" | "thinking" | "speaking" | "listening";
interface AgentAdapter {
  readonly id: string;              // "none" in v2; later "claude" | "codex" | "ollama" | …
  readonly connected: boolean;
  ask(q: { text: string; attachments: readonly File[] }, signal: AbortSignal):
    AsyncIterable<{ state: AgentState } | { markdown: string }>;
}
```

- v2 ships only the `none` adapter. It yields `thinking` briefly, then one honest message: "No agent
  is connected to ASK HQ yet." It is client-side, so v2 adds **no** server route for ASK HQ.
- The shell knows nothing runtime-specific.

**The spectrum, Variant B · Mirror only.** A and C and the demo pill are removed.
- 64 log bands, 80 Hz–4 kHz.
- The source is the voice model (f0 ≈ 132 Hz with prosody, syllables at 3.6 Hz, 5 formants, 16
  harmonics) or the mic FFT (8192).
- Attack/release `.55/.18`, or `.22/.07` while speaking. Peak hold 18 frames, then a fall of .012
  per frame.
- Mirror: 200 bars, 3 px with round caps. Each bar reads an fbm-chosen band, has its own gain
  (0.45–1.4) and rides a two-scale noise envelope.
- Spikes: about 2.5 % of bars at 8 Hz when speaking, about 4 % at 14 Hz when listening.
- Reach: out `6+78·amp` and in `3+26·amp` when speaking; in `6+66·amp` when listening.
- Thinking is noise clusters drifting and swelling at their own speeds, rare sparks, 3 px hairlines.
- Speaking radiates out in ember. Listening draws in in gold, with three contracting rings, and its
  brightness follows the mic level.
- The dust cap pulses while speaking and turns gold while listening.
- The spectrum and noise maths (band mapping, attack/release and peak hold, fbm, the voice model
  and the per-bar parameters) are pure, seeded functions of `(seed, t, level)`.

## Testing Decisions

- **A good test checks behaviour at a seam.** Fixtures go in; a graph, an HTTP response, a recorded
  argv list or a pure value comes out. Tests never assert pixels, DOM structure beyond roles and
  text, or private helpers.
- **Seam 1: the HTTP surface** (extended), built with injected dependencies (graph, readers,
  executor, clock, ledger sink). Tests cover:
  - every non-GET except `POST /action` is still 405 (the v1 suite, kept);
  - the origin guard: each of these gives 403 and is ledgered:
    - a wrong or missing `Origin`;
    - `Sec-Fetch-Site: cross-site`;
    - a non-JSON content type;
    - a rebinding pair, `Host: evil.test:<port>` with `Origin: http://evil.test:<port>`;
  - unknown node id → 404; a verb outside the kind's list → 403; an M1 routine → 403;
  - HQ's own label (`com.silou.hq`) refuses every control verb;
  - **no request can name a path**: a `path` in the payload is ignored, and `script` writes only to
    the plist-resolved script;
  - a plist whose only argument is an interpreter (`/opt/homebrew/bin/python3`) has no editable
    script, so `script` gives 403;
  - `restore` on `routines:removed/<label>` works after a rebuild that no longer contains the
    routine's node;
  - each verb's exact argv and file writes, as recorded by the executor;
  - `create` validation (label pattern, the `repo` id, refusing an existing plist or script, 409);
  - the size cap; snapshot-before-write; `restore` consuming the newest snapshot; 409 when there is
    none;
  - per-label serialisation (409);
  - ledger lines for done, refused and failed, with script content never in the ledger;
  - `/routine?id=` allowlisting and bounded tails; `/ledger.json` limited to 200.
- **The guard test is rewritten.** v1's grep-tested "the server branches on no write method except
  to refuse it" becomes "the server branches on exactly one write method and path, `POST /action`".
- **Seam 2: pure modules.** Tests cover:
  - the verb allowlist (table-driven, every kind);
  - the action catalogue (`sectors()` per kind and per routine state: running, disabled, broken,
    M1, has-snapshot);
  - the schedule codec: round-trip plist ↔ schedule for every kind, inexpressible plists flagged,
    the sentence and next run at fixed clocks, the 5-minute snap; it shares fixtures with hq-10;
  - routine health from fixture `launchctl` outputs (consistent with hq-12);
  - script resolution from fixture `ProgramArguments`;
  - the hover state machine: scripted event sequences for magnet hold and release, near-miss click,
    the ghost's 750 ms stillness with resets, promote, and hover-close at 300 ms;
  - bezel geometry: sector bounds, orbit radii, upright-text flags, LINKS paging and the turn
    accumulator, the focus-zone radius easing;
  - ring layout additions: the always-on arc, the plugins·mcp track, instruction docking, the
    focused clamp and the 8–92 % pan bound, the UI scale;
  - the LCS diff and its counts; the ×N log fold;
  - the spectrum and noise functions: deterministic for a seed, values bounded, irregular (no two
    adjacent bars share a band, gains vary), and the attack/release and peak-hold curves.
- **UI** (happy-dom, Preact, one component per file, as in v1). Each surface is tested at its empty
  and offline states:
  - the console with no routines and with launchd unreadable;
  - the editor for a broken routine (template pre-filled) and with no changes (save bar hidden);
  - the save bar's inline confirmation flow;
  - the M1 routine read-only;
  - ASK HQ with the `none` adapter (honest message, no fake reply).

  The canvas is not unit tested beyond its pure maths.
- **Prior art:**
  - silou-hq's own `test/serve.test.ts` (refusal, allowlist and tail hardening suites);
  - `test/web/*` (happy-dom components);
  - `test/launchd/*` (plist parsing);
  - `rings-layout` tests from hq-09;
  - adw-factory's `test/web/*` and its grep-tested route guard.
- **Gates:** `bun run lint && bunx tsc --noEmit && bun test`.
- **Manual checks the factory can't run,** listed in the relevant tickets for the operator: trackpad
  pinch and pan, the real mic, animation timing in a visible tab, the dial drag, and one real
  run/pause/resume on a throwaway routine.

## Out of Scope

- **The ASK HQ agent backend:**
  - which runtime or model answers;
  - what context it gets (the whole graph?);
  - whether it may trigger verbs;
  - speech-to-text (browser STT may send audio off the machine);
  - any server route for it.

  It gets its own spec, which will amend the write surface again if it needs a route.
- **Control of M1 routines** (shown read-only).
- **Write verbs on anything but routines:** running skills, editing memory, factory dispatch, and
  hooks.
- **A lease or write-mode switch** (decided against).
- **The smoked-glass focus material and the `?lens` A/B,** and ASK HQ variants A · LED and
  C · Ribbons.
- **Prototype chrome:** the think/speak/listen demo buttons, the per-request bundle rebuild, and the
  `/proto/*` routes.
- **Scheduling shapes beyond the four editor kinds** (shown read-only).

## Further Notes

**Where this spec deliberately departs from the prototype:**
- The prototype's `script` verb took a path from the client. v2 resolves it from the plist (rule #4).
- The prototype faked every write. v2 executes, with the executor injected for tests.
- The prototype's "Runs in" was free text. v2 picks from repo nodes, and the script lands in
  `<repo>/routines/`.
- The prototype had no origin guard, because it had no real writes.
- Routine health extends hq-12 instead of the prototype's separate computation.

**Decisions made while writing this spec, which the operator can veto at review:**
- the UI scale formula `clamp(w/3000, 0.7, 1)`;
- the always-on arc at 6 o'clock, at most 40°;
- the previous-version cap of 10 per label;
- the ledger view limit of 200;
- the `create` label pattern.

**Related tickets:**
- **hq-12** (done) defines "failed". The console builds on it.
- **hq-13** (M1 routines report their failures, blocked) is independent; the console shows whatever
  M1 health exists.
- **hq-15** (queued) and **hq-16** (queued) are independent.

**Live first story:** brain-hub's `d1-channels`, `d2-redshift` and `d3-snapshot` routines still
fail daily ("No such file or directory", ×120 in the tail). Recreating their scripts from the
console is the v2 acceptance demo.

**Known prototype rough edges, carried as ticket checks:**
- animation timing was never verified in a visible tab;
- the first click after load is lost (story 74);
- a click during the magnet hold selects the held node (intended; watch it).

**Rejected, do not rebuild** (PROTOTYPE-NOTES §8):
- the v1 Rim and Dock rings;
- the magnifying loupe;
- floating action chips;
- the camera dive on ASK HQ;
- the page-wide dim;
- a solid disc behind the bezel;
- borders on ring-focus bands;
- the centre as "what needs you" or a quick-actions hub;
- octagonal centres;
- a blue ASK HQ;
- a symmetric spectrum;
- the LED "chase" thinking.

**After approval:**
- tracer-bullet tickets go in `~/adw/backlog/silou-hq/` (the central store);
- the prototype worktree `~/personal_project/silou-hq-proto-ring` can then be removed. Ask first.
