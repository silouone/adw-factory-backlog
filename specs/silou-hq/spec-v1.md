# silou-hq — v1 spec

> **Status:** agreed 2026-10-01. It came out of three grilling rounds plus the follow-up
> questions Q17 and Q18, and the operator-approved v2 prototype (formerly
> `adw-factory/.proto/hq-os`).
> **Amended 2026-10-03 by `docs/spec-v2.md`:** read-only is replaced by local, allowlisted, ledgered writes (see its Amendment section).
> **Amended 2026-10-03 by `docs/spec-v3-mail-calendar.md`:** rule #2 now allows provider credentials (Keychain only) and rebuildable caches; mail and calendar sources replace the empty states (story 41).
> **Working name:** silou-hq. It is a factual placeholder; the real name comes later.
> **Labels:** `ready-for-agent`.
> **Background reading** (adw-factory `ai_docs/`, local only):
> - `2026-10-01-supervisor-and-hq-continuation.md`
> - `2026-10-01-hq-adhd.md`
> - `2026-10-01-ai-os-landscape.md` (plus its slices A and B)

## Problem Statement

My working setup is spread over two Macs, ~190 repositories, ~600 memory files, ~120 skills
across three agent runtimes, launchd routines on both machines, and a software factory with its
own dashboard. To answer "what is going on, and what needs me?" I currently need several
terminals, `just status`, the adw web board, and the memory folders, opened one after another.

Nothing shows the whole setup at once. Nothing shows which of my skills would run on a runtime
other than Claude. Nothing shows whether my scheduled routines actually ran. On the day this spec
was written, a brain-hub routine had been failing every day with "No such file or directory", and
nobody had noticed.

The factory is good at its own job: a ticket → PR kanban. It is not, and should not become, the
place where I see everything else.

## Solution

A local, read-only landing page: **the HQ**.

**The rings.** At its centre is my whole setup, drawn as concentric rings **grouped by type**,
from the centre outwards:

| Ring | Contents |
|---|---|
| core | the instructions every agent reads (CLAUDE.md, AGENTS.md) |
| RUNTIMES | any LLM is a tool: claude code, codex, ollama, plus MCP servers and plugins |
| SKILLS | one library merged across machines and runtimes; portable skills first |
| MEMORY | memory files plus configured knowledge folders, in sectors aligned with the project clusters |
| ROUTINES | a 24-hour dial for both machines; hooks shown as event triggers |
| PROJECTS | the clusters: adw-factory, brain-hub, sabado, go1, coorp, setup, lab, with their repos on the outer arc |

**The widgets.** Movable, resizable widgets surround the rings:

| Widget | Shows |
|---|---|
| Today | clock, quarter-week grid, next routines |
| Factory | summary of the factory's work |
| Projects | one row per cluster |
| In progress | running now, waiting on you, needs you (failed routines), ready to review, blocked |
| Routines | timetable, with each routine's latest output |
| Skills deck | recent, portable and Claude-only skills |
| Mail | not connected yet |

**Full screen.** Every quick access opens full screen: the factory's own dashboard (embedded), a
project, a routine's output, the skills library, or any file.

**Global search.** `/` searches across everything.

**Plugged-in tools.** The factory, and later brain-hub, are tools plugged into the HQ through
their public surfaces. They are never coupled to it.

## User Stories

**The landing**
1. As the operator, I want one page that shows my whole setup, so that I stop opening five tools to answer "what's going on".
2. As the operator, I want the setup grouped by type in rings (core, runtimes, skills, memory, routines, projects), so that I never see a runtime next to a project.
3. As the operator, I want the landing to appear in under a second, so that it is the place I go first.
4. As the operator, I want to zoom with a bounded, slow, eased wheel and pan within the rings, so that I cannot lose the picture or zoom out into empty space.
5. As the operator, I want clicking any item to glide the camera there with a short trail from the previous item, so that I keep my bearings.
6. As the operator, I want to double-click or press Esc to return to the fitted view, so that I can always reset.
7. As the operator, I want hovering an item to show its kind, cluster and one key fact, so that I can scan without clicking.
8. As the operator, I want a selected item's connections lit and everything else dimmed, so that I can see what it touches.
9. As the operator, I want each cluster's badge on the outer ring to show its open-ticket count with a live, waiting or blocked dot, so that I can see where work stands at a glance.
10. As the operator, I want the repos of a cluster to expand along its arc only when I zoom in or select it, so that 190 repos never clutter the fitted view.
11. As someone who prefers reduced motion, I want every animation off when my system asks for it, so that the page stays calm.

**Search and inspection**
12. As the operator, I want `/` to open a search across every kind of item, grouped by kind, so that I can find a skill, a memory or a repo in seconds.
13. As the operator, I want Enter to select the first hit and fly to it, so that keyboard search is enough.
14. As the operator, I want an inspector that shows an item's kind, cluster, machines, description, connections and a preview of its file, so that I rarely need to leave HQ.
15. As the operator, I want to open any file-backed item (memory, report, spec, skill, hook, core instructions) full screen, so that I can read it properly.

**Skills, any LLM**
16. As the operator, I want one skills library merged by name across `~/.claude`, `~/.agents`, `~/.codex`, plugins and the M1, so that duplicates collapse into a single entry.
17. As the operator, I want each skill marked with the runtimes it runs on and the machines it lives on, so that I can see which skills are portable and which are Claude-only.
18. As the operator, I want the skills deck filtered by recent, portable and Claude-only, so that I can focus on what still needs to become LLM-agnostic.
19. As the operator, I want a full-screen skills library with filters (all, portable, claude-only, codex-only, m1) and a SKILL.md reader, so that I can browse the whole library.
20. As the operator, I want the runtimes shown as interchangeable tools, so that the setup reads as "any LLM can run this".

**Memory and knowledge**
21. As the operator, I want every memory file from both machines shown in its cluster's sector, so that I can see where knowledge lives.
22. As the operator, I want memory that exists only on the M1 visually distinct, so that I can triage what to migrate.
23. As the operator, I want configured knowledge folders (the factory's reports and specs) included, so that design documents sit beside memory.
24. As the operator, I want links between memory files, skill mentions and report references drawn as connections, so that related knowledge shows up together.

**Routines**
25. As the operator, I want every launchd routine on both machines on a 24-hour dial with a "now" hand, so that I can see today's rhythm.
26. As the operator, I want a routines timetable marking each routine as earlier today, next, queued or always on, with the machine it runs on, so that I know what runs when.
27. As the operator, I want each routine's latest output (stdout and stderr tail) one click away, full screen, so that a failing routine is visible the day it fails.
28. As the operator, I want hooks listed as on-event triggers, so that automation that isn't on a schedule is still visible.
29. As the operator, I want the last 24 hours of activity ticked on the dial, so that I see when things happened.

**Projects and work**
30. As the operator, I want each cluster to count its repos, memories, open tickets, blocked tickets, items in review, live runs and live sessions, so that every project has a one-line health check.
31. As the operator, I want a project page full screen (work, recent runs, sessions, routines, memory, repos), so that I can get oriented in one project without the factory's kanban.
32. As the operator, I want an "In progress" widget (running now, waiting on you, needs you (failed routines), ready to review, blocked), so that I know what needs me today.
33. As the operator, I want live Claude sessions shown, with the one waiting on a dialog highlighted, so that I never leave a session stuck.

**The factory, plugged in**
34. As the operator, I want a factory widget with running, queued, blocked and to-review counts and a strip of recent runs, so that I can read the factory's pulse without opening it.
35. As the operator, I want the factory's own dashboard to open full screen inside HQ, so that a deep dive is one click and Esc brings me back.
36. As the operator, I want HQ to say plainly when the factory is not configured or not reachable, so that a stopped `adw web` never breaks HQ.
37. As the factory's maintainer, I want HQ to read only the factory's public JSON, so that factory internals can change without breaking HQ.

**Widgets and layout**
38. As the operator, I want to move and resize widgets in an explicit arrange mode, snapped to a grid, so that dragging a widget never fights with panning the rings.
39. As the operator, I want my layout remembered and resettable, so that the page is mine.
40. As the operator, I want the default layout to follow the window until I arrange it, so that a new screen size never hides a widget.
41. As the operator, I want honest empty states for calendar and mail ("not connected yet"), so that missing sources are never faked.

**Safety, operations**
42. As the operator, I want HQ to be strictly read-only in v1, refusing every non-GET request, so that it can never change my setup by accident.
43. As the operator, I want files and logs served only for items HQ already knows, never by a path, so that HQ cannot be used to read arbitrary files.
44. As the operator, I want the M1 read over SSH with a timeout and a cached snapshot, so that an offline M1 never blocks the landing.
45. As the operator, I want HQ to keep running in the background under launchd on this Mac, so that it is always one bookmark away.
46. As the operator, I want no private data (snapshots, caches) committed to git, so that the repo is code only.
47. As the operator, I want clusters, knowledge folders, the factory's URL setting and the M1 host in one config file, so that changing my setup means editing data, not code.

## Implementation Decisions

**Repo and process**
- **Repo:** its own repository next to adw-factory. It is local only for now; the operator adds the remote right before the first factory ticket. The factory builds it as target `silou-hq`.
- **Host:** this MBP. One Bun process serves the page and the JSON. A launchd agent with `KeepAlive` and a fixed port keeps it up. Moving to the M1 or a VPS is a later decision.
- **Read-only, absolutely:**
  - no write route;
  - every non-GET request gets 405;
  - v1 has no action buttons.
  - Triggering skills and routines is **v2**.
  - HQ owns no loops or schedule. Its "detectors" (a broken routine (failed, per *Routine health*), a waiting session, blocked work) are computed on every refresh.

**Data**
- **HQ owns no data.** Every view is derived from files and public surfaces. The only state HQ keeps is:
  - the M1 snapshot plus caches, in a gitignored `cache/` folder;
  - the widget layout, in browser storage.
  - SQLite is deferred and only considered if the 1-second target is missed.
- **Routine health (this Mac):** on each local refresh HQ runs `launchctl list` (read-only) and keeps, per owned label, the PID and the last exit status. A routine is **failed** when it has no PID and its last exit status is non-zero. It is **not** failed when the status is `-` or `0`, or when it currently has a PID (a KeepAlive job that was restarted, such as `com.user.brain-hub-mcp`, running with a last status of 143). **The M1** follows the same rule, from the PID and last exit status in its snapshot. An M1 that is offline, a snapshot older than `2 × scanEveryMin`, or a snapshot from an older scanner (no health fields) gives **unknown**, never failed.
- **Speed:**
  - the landing renders from the last built graph;
  - rebuilds happen in the background (local: every 60 s; factory: every 30 s; M1 snapshot: every `scanEveryMin`, 15 by default);
  - slow sources never block a request.
- **The M1:** `ssh <host> /usr/bin/python3 - < scanner` with a 60 s kill. It reads only:
  - repo names (depth 3, excluding Library and media folders);
  - memory frontmatter;
  - skill frontmatter (claude, codex, agents);
  - launchd plists HQ owns (`com.silou*`, `com.user.*`, `io.sabado*`);
  - the PID and last exit status of the launchd labels HQ owns (`launchctl list`, filtered to the same prefixes).

  Output goes to `cache/m1-snapshot.json`, which stays the M1 source until the operator triages that memory.
- **Configuration** (one JSON file):
  - port;
  - M1 host and scan interval;
  - factory name, the env var holding its URL (`ADW_WEB`), and a project → cluster map;
  - knowledge folders (dir, kind, cluster);
  - clusters (id, label, what, match regex), with the catch-all last.

**The graph model** (shape from the prototype). Node kinds:

| Group | Kinds |
|---|---|
| core | `core` |
| runtimes | `runtime`, `mcp`, `plugin` |
| skills | `skill`, `command`, `subagent` |
| memory | `memory`, `report`, `spec` |
| routines | `routine`, `hook` |
| projects | `cluster`, `repo` |
| live | `session` |

Each node carries:
- `cluster`, `machines` and `sources`;
- `meta` (cluster counts, routine schedule, skill portability);
- `degree`, `hasFile` and `hasOutput`.

Links are undirected `{source, target}`. Events are `{at, kind, label, nodes[]}`. The graph also
carries factory `work`, `live` and `recent`, plus `m1` and `factory` status.

**Skills and portability**
- Skills are merged by name.
- **portable** = available in `~/.agents`, or on both Claude and Codex.
- **claude-only** / **codex-only** = the rest.
- Each skill links to every runtime that can run it. v1 only shows this; it never moves a skill.

**Clusters**
- Every repo, memory directory and session cwd maps to a cluster by the first matching regex in config.
- Factory projects map through the config's project → cluster map.
- Factory worktrees are excluded from repos.

**The factory contract**, the only coupling:
- HQ reads `GET /backlog.json` (projects[].rows: id, title, status, kind, closed).
- HQ reads the first frame of `GET /events` (views[]: runId, ticketId, target, state ∈ running/finished/unknown, outcome, durationMs, reason).
- Both are read from the URL in the configured env var, token included. The run time is derived from the factory's documented `<ticketId>-<epochMs>` runId.
- The factory's dashboard is embedded full screen.
- HQ never reads the factory's `runs/`, `targets/` or ticket store.
- If the factory is unreachable, the widget says so and HQ keeps working.

**Allowlisted reads**
- `/file?id=` and `/output?id=` resolve only ids recorded in an allowlist that the graph build fills.
- Results are a bounded tail (400 and 120 lines respectively).
- M1 entries are read with `ssh tail` under a timeout.

**UI**
- One canvas draws the rings: no 3D library, eased camera, k ∈ [1, 4.5], pan clamped.
- Widgets are DOM over the canvas, in an explicit arrange mode.
- Full screen is one overlay with five kinds: factory, project, output, library, file.
- **v1 ports the widgets, search, inspector and full screen to Preact + signals** (the same stack as adw web). The rings stay on canvas.

**Look:** adw-factory's theme tokens, plus one muted hue per ring and per cluster. `ui-monospace`
for body text and data; the display face only for titles. Thin strokes, no glow.

## Testing Decisions

- **A good test checks behaviour at a seam,** not internals. It goes in fixtures and comes out as a graph, an HTTP response or a rendered widget.
- **Seam 1: `buildGraph(inputs)`**, pure. Inputs are injected readers (a fixture home folder, a fixture M1 snapshot, fixture factory backlog and events frame, fixture `claude agents` output). Tests cover:
  - cluster mapping and the catch-all;
  - skill merging and portability;
  - routine schedules (interval, calendar, weekday, always on);
  - cluster counts;
  - factory mapping and the unreachable path;
  - worktree exclusion;
  - memory machine marks;
  - the allowlist contents.
- **Seam 2: the HTTP surface**, built with injected dependencies. Tests cover:
  - 405 on every non-GET;
  - 404 for ids not in the allowlist and for path-shaped ids;
  - bounded tails;
  - `/factory.json` honesty (not configured vs not reachable).
- **UI:** the Preact widgets get happy-dom tests for their empty and offline states (factory not reachable, calendar and mail not connected). The canvas is not unit tested; its pure layout maths (angles, sectors, clamp) is, if extracted.
- **Prior art:** adw-factory's `test/web/*` (happy-dom, a Preact component per file), its pure-projection tests (`backlog-graph`), and its grep-tested "no mutating route" guard. That guard is re-used here as a test that the server source branches on no write method except to refuse it.
- **Gates:** `bun run lint` (Biome, warnings allowed), `bunx tsc --noEmit` (strict), `bun test`.

## Out of Scope

- **Every write action:**
  - running a skill;
  - triggering, pausing or editing a routine;
  - dispatching, killing or resuming factory runs;
  - editing memory or marking it verified.

  All of this is v2, behind a lease and an action ledger.
- **HQ's own scheduled loops** (`hq.tick`, `hq.detect`, `hq.sleep`).
- **Making skills portable** (moving them into `~/.agents`). v1 only shows it.
- **Mail and calendar sources:** empty states only.
- **brain-hub integration:** it appears as a cluster, and its dashboard or MCP plug-in comes later.
- **Hosting on the M1 or a VPS; multi-user; authentication.**
- **SQLite or any database.**
- **A remote git repository.** The operator adds it before the first factory ticket.

## Further Notes

**Found on first run** (2026-10-01):
- brain-hub `d2-redshift` fails daily (its script is missing).
- Three `adw web` processes were running.

**Factory-side follow-ups**, filed in adw-factory's own backlog: a stable `adw web` URL and token
(today they change on every launch, which makes `ADW_WEB` a manual copy), and the `silou-hq`
target plus its ticket store.

**Reversals recorded:**
- July's "dashboards and an always-on heartbeat aren't worth it" is reversed for HQ, not for the factory.
- "HQ in the factory repo" is reversed, because the scope outgrew the factory.
