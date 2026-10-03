# Amendment v1.2 — the live factory view (`adw web`)

> **Status:** proposed 2026-09-11 after an operator grilling session (three
> rounds; every decision locked below). **Amends:** `adw-v1.md` §2 non-goals —
> "Dashboards or new visualization surfaces". **Binding:** `constitution.md` —
> unchanged. **Design record:** `ai_docs/2026-09-11-live-factory-view-reflection.md`.
> **Depends on:** nothing. **Benefits from:** `adw-par-01-repo-commit-lock`
> (without it the grid has at most one live cell).
> **Amended 2026-10-02 (adw-web-01):** the token is *per-launch by default,
> operator-fixable* — `ADW_WEB_TOKEN` / `ADW_WEB_PORT` let a tool that embeds
> the view keep one address across restarts. There is deliberately no
> `--token` flag (argv leaks via `ps`); `--port` beats `ADW_WEB_PORT`; an empty
> value means unset. Loopback bind, the token on every request, and
> read-only-by-construction are unchanged.
> **Deliberately excluded:** `adw-sysprompt-01-own-the-system-prompt` — the UI
> reserves a slot for the system prompt; *choosing* its value is that ticket's
> job, not this one's.

## Trigger (why this is allowed, not a silent diverge)

`adw-v1.md` §2 defers dashboards with an explicit promise:

> *"Dashboards or new visualization surfaces (the data is banked from run #1;
> views come later)."*

The deferral was conditional on the data being banked, and it is: **67 runs with a
`journal.jsonl` and OTel spans; 54 of them with cLens captures, carrying
2 256 `PreToolUse` records** (counted 2026-09-11 against `runs/`).
Every figure in this spec is **dated, not perpetual** — `runs/` is gitignored,
so `scripts/run-metrics.ts` is the reproducible path back to these numbers. This amendment is the "later" the non-goal
named. It un-defers **views** only. A router agent, a watcher daemon,
scheduled runs and queue draining all remain deferred.

Two Art. VI ("observable by construction") debts are settled here as a
side effect, and both are worth having **even if no FE is ever built**:

1. **The factory cannot tell a dead run from a working one.** No heartbeat,
   no pid. The `adw-m5-06` run log records the cost in the operator's own
   words: a run *"died hard… the ticket was left `in-progress` with
   `attempts: []`. Reset to `queued` by hand."*
2. **The factory discards most of what it knows about its own agents.** The
   resolved model, tool list, MCP servers, skills, plugins and CLI version all
   arrive on the SDK `system/init` message and are dropped one line into the
   consumer. Every compiled prompt is held in memory and never written.

## Problem Statement

The operator dispatches a ticket and then goes blind. A run takes 2–90
minutes, and for its entire duration the only available signals are a
scrolling terminal and, afterwards, `just status` — one line per ticket.

Concretely, today the operator cannot answer:

- Is this run still alive, or did the process die twenty minutes ago?
- Which node is it in, and how long has it been there?
- Is the agent doing anything, or is it wedged?
- What did the last eleven runs have in common when they blocked?
- What prompt was this agent actually given? *(Recoverable only by hand, from
  a 30 KB field inside a capture JSONL — and absent entirely for 18 of 69
  captured sessions.)*
- What model, tools and configuration did it run with? *(Not recorded at all.)*

The factory has been observable-by-construction since M3 in the sense that it
**writes** its evidence. It has never been observable in the sense that a human
can **look** at it while it happens.

## Solution

A read-only local web view, `adw web`, serving two screens over the artifacts
the factory already writes:

1. **The run grid** — one card per run, newest first: ticket id, target, lane,
   isolation kind, provider, outcome, duration, and a compressed per-role
   lane strip. Live runs sort first and update in place.
2. **The run Gantt** — roles as horizontal rows, time as the x-axis, each node
   a block positioned and sized by its real duration, tool calls rendered as
   ticks inside the block. A left rail per role carries its model. Selecting a
   block opens a drawer with that node's owner, kind, attempt, gates, and its
   compiled prompt.

Plus the two factory changes that make those screens honest: a **heartbeat**,
so "running" can be wrong; and **unconditional persistence** of agent prompts
and configuration, so the drawer has something true to show.

**What this is not.** Not a fleet monitor (the factory is manually dispatched;
concurrency is the operator's, from separate terminals). Not a control plane —
it never acts on a run. Not a second cLens — session interiors remain cLens's
job, reached by launching it, not by an embedded viewer.

## The change

### A. Persistence — the factory records what it knows (Art. VI)

Unconditional, independent of any FE. Three tiers, one seam (the agent query
boundary):

- **Free.** Widen the SDK message mirror so the `system/init` message is
  retained rather than reduced to `session_id`: resolved model, tool list,
  MCP servers + status, skills, plugins, slash commands, permission mode,
  CLI version. Journaled inline — it is small.
- **Cheap.** Write every prompt actually handed to an agent. This is **four
  sites**, not one: the assembled prompt, plus the three built inline and
  never placed on run context — `repair` (which carries the full diff so far),
  `ci-repair`, and `revise`.
- **Extra.** The allowed-tools list, the agent environment, and which prompt
  template file was selected.

**Two additions to `AgentUsage`, both required by the metrics in §D and both
data the factory currently throws away at the seam:**

- **`model`** — the resolved model per role. Today `provider`/`model` reach
  only the ticket's `attempts[]` at finalize: **per run, not per role, and not
  live.** Without it the left rail cannot be labelled and no per-model rate
  can be computed.
- **The usage BREAKDOWN, not the sum.** `build.ts:472` adds `input_tokens +
  output_tokens` and discards everything else — including **cache reads, which
  on a cached run are the large majority of the real prompt**. Retain the four
  components.

**Context occupancy needs both, and still needs a third thing we do not have.**
The convention (cLens's own `distill/context-consumption.ts`, and SSSF) is the
last valid assistant turn's `input + output + cache_read + cache_write` —
cached prompt is still prompt. That is the numerator, unlocked by the
breakdown above. The **denominator** is the model's context window, which needs
a model→window map; cLens carries one but only inside `distill`, i.e.
**post-hoc, not available live**.

**Decision:** if that map proves contentious, ship the rail as an **absolute
token count rather than a percentage**. A real number beats a percentage of a
guessed denominator.

**Storage is hybrid, and the split is load-bearing.** Prompt bodies run 4–30 KB
(median ~16 KB) and a `feat` run compiles three; inlining them would make the
journal an order of magnitude larger and force the view's tailer to stream
megabytes of text it does not render. So: **the journal stays the index** — it
already is, by Art. VIII — gaining a record that names the node, the sidecar
path, the byte length and a content hash. Bodies land as sidecar files under
the run directory. Small configuration stays inline.

**Note the existing defect this fixes.** The `feat` lane compiles three
distinct prompts and every one overwrites the same run-context key; only the
last survives to run end. Persisting at the query boundary — where each prompt
is actually used — is what makes all three recoverable.

### B. Heartbeat — "running" must be able to be wrong (Art. V)

A periodic journal event on the engine's existing timer seam, which is
single-shot, so the heartbeat is a self-rescheduling chain: arm 15 s → append
→ re-arm. Cancelled on run-end, on abort, and on every terminal path.

- **Unref'd**, matching the established precedent. A pending heartbeat must
  never by itself keep the lane alive — otherwise the mechanism meant to
  detect a wedged run would be what makes it look healthy.
- **Bare payload.** The current node is already derivable from a `node-start`
  with no matching `node-end`; restating it would be duplicated derivable
  state (Art. VIII).
- **Engine-owned**, like the existing journal middleware, so a node can no more
  run un-heartbeated than it can run unjournaled.

**Staleness: 60 s — four missed beats.** Beyond it a run renders `unknown`,
never `running`.

**Three signals, three renders, and the view must not conflate them:**

| Signal | Proves | Render |
|---|---|---|
| heartbeat stale | the lane process is gone | run → `unknown` |
| `watchdog` event *(exists today)* | one agent went silent past tolerance | that lane stalls; run is fine |
| — none — | alive but looping | **not detectable in v1; the view must not imply otherwise** |

Rejected alternatives, both cheaper and both wrong: *journal file mtime* fails
exactly where it matters, because a 40-minute build node appends nothing;
*capture file mtime* reports the healthiest runs as dead, since deterministic
nodes produce no capture by construction and **18 of 69 captured sessions have no recorded prompt at all** (2026-09-11).

### C. `target` on run-start

`runs/` is one flat directory and the journal never records which target repo a
run was against. Without it the grid mixes self-hosted factory runs with cLens
product runs indistinguishably. One string, on the event that already exists.

### D. The projection — one pure function

`(journal records, capture records) → run view`. Every rule below lives here
and nowhere else, and it performs no I/O:

- **Lane assignment.** Lanes derive from what is already journaled: the node
  end detail discriminates agent nodes from gate nodes from plain
  deterministic ones, and **for agent nodes the node name is the role**
  (`plan` → planner, `build` → builder, and so on). Deterministic nodes share
  one `workspace` lane; gate results render as chips within it.
- **Dot attribution binds a tool call to a lane by `timestamp ∩ node window`
  — never by session id.** Session id is **1:N, not 1:1**: repair and CI-repair
  *resume* the original session, so their tool calls append to the same capture
  file. Attributing by session id alone would render repair activity as builder
  activity. This rule is cheap now and expensive to retrofit.
- **⚠ Capture sessions are addressed BY THE JOURNAL, never by globbing the
  directory.** `runs/<runId>/.clens/sessions/` is resolved by a filesystem
  walk-up, so **any process whose cwd is inside the run's workspace captures
  into it** — including the operator's own shell. This is not hypothetical: on
  2026-09-12 an operator `bun test` run during the salvage of
  `adw-par-01` landed a 405-second command in that run's capture dir and was
  briefly attributed to the agent. The projection MUST use only the session ids
  the journal names in its `usage` / `capture` events; anything else in the
  directory is foreign and is ignored.

- **Three run states** — `running` / `finished` / `unknown`.
- **Per-node and per-role metrics** — each derived, none stored:

  | Metric | Derivation | Why it earns its place |
  |---|---|---|
  | **duration** per node | `node-end.ts − node-start.ts` | already the Gantt's block width |
  | **turns** per node | `usage.turns` | the budget that actually terminates runs |
  | **tokens** per node | `usage.tokens` | |
  | **seconds/turn** | duration ÷ turns | **the single most diagnostic number** — see below |
  | **tokens/second** | tokens ÷ duration | throughput per role |
  | **tool time vs model time** | Σ `PostToolUse.duration_ms` vs (duration − that) | separates a slow suite from a slow model |
  | **context %** per role | occupancy ÷ the model's window — **neither input exists today, see §A** | the `CONTEXT 3%` rail |
  | **model** per role | `AgentUsage.model` — added in §A | the left-rail label |

  **`tokens` is an undercount and the view must label it as one.** The journal's
  `usage.tokens` is `input + output` summed at `build.ts:472`; the split is
  discarded and **cache reads are not counted at all**. On a cached run that is
  the large majority of the real prompt. Render it as "billed in/out", never as
  "tokens used", and derive tokens/second from the same figure so the two agree.

  **Why seconds/turn is load-bearing.** On the `adw-par-01` run it read
  `plan 18.1s · build 6.6s · test 6.9s` — and the tool/model split showed
  `plan` was **99% model, 1% tool** (33 Bash calls averaging 0.2s). That single
  pair of numbers distinguishes "the model is generating a long document"
  from "the test suite is slow" from "the agent is thrashing", and **no
  existing surface can answer it.** It is the reason this view is worth
  building beyond liveness.

  **Blocked-on-subagent time must be visible.** The same run spent **12.1 of
  the `test` node's 23 minutes** inside three blocking `TaskOutput` waits, and
  because subagent turns are deliberately excluded from the budget
  (`build.ts`, `parent_tool_use_id === undefined`) that time was **invisible in
  every existing metric** — it did not appear in the 422-turn count that
  ultimately killed the run. Render it as its own band, not as model time.

- **Capture health is a distinct state from silence.** "No capture recorded"
  and "captured, zero tool calls" are different facts; **18 of 69** sessions have
  the former. Conflating them is the same class of lie as `unknown`-as-`running`.

### E. The server edge

`Bun.serve` + Server-Sent Events + a byte-offset journal tailer. A thin shell
over D, with its filesystem, watch and clock dependencies injected like every
other edge in this codebase.

- **Zero new dependencies.** Two screens of positioned rectangles and a drawer
  do not need a framework, and every dependency added here is a dependency in
  the factory's own manifest forever (Art. VIII).
- **Append-only JSONL is the cursor.** No database, no write-ahead log, no
  row-id bookkeeping — the file offset *is* the read position.
- **Loopback bind plus a token** — per-launch by default, operator-fixable via
  `ADW_WEB_TOKEN` (and `ADW_WEB_PORT`; `--port` wins) — printed to the terminal. The view
  serves prompt bodies containing ticket text, repo context and diffs; this is
  the posture cLens already chose for the same data on the same machine.

## User Stories

1. As the operator, I want to see whether a dispatched run is still alive, so that I stop discovering hard-killed runs by hand hours later.
2. As the operator, I want a run that has stopped heartbeating to render as `unknown` rather than `running`, so that the view never tells me a comfortable lie.
3. As the operator, I want to know which node a live run is currently executing, so that I can judge whether its elapsed time is reasonable.
4. As the operator, I want to see how long the current node has been running, so that I can decide whether to intervene before a ceiling trips.
5. As the operator, I want each role on its own row against a real time axis, so that I can see at a glance which stage dominated a run.
6. As the operator, I want node blocks sized by actual duration, so that a 40-minute build and a 4-second dispatch are visually honest.
7. As the operator, I want individual tool calls rendered inside a node's block, so that I can distinguish a working agent from a wedged one.
8. As the operator, I want repair-round tool calls attributed to the repair lane and not the builder lane, so that the timeline reflects what actually happened.
9. As the operator, I want a run with no capture to say so explicitly, so that I never read a missing recording as an idle agent.
10. As the operator, I want a watchdog trip to stall one lane while the run keeps rendering as healthy, so that agent silence and process death are visibly different failures.
11. As the operator, I want a grid of every past run, so that I can spot patterns across runs instead of reading journals one at a time.
12. As the operator, I want live runs sorted first in the grid, so that the thing happening now is never below the fold.
13. As the operator, I want each card to show its target repo, so that factory self-runs and product runs are never confused.
14. As the operator, I want each card to show its isolation kind, so that I can tell a free worktree run from a billed remote one at a glance.
15. As the operator, I want each card to show its provider, so that Claude and Codex runs are distinguishable without opening them.
16. As the operator, I want each card to show its lane, so that I can compare chore, bug and feat runs as groups.
17. As the operator, I want to see outcome and duration on the card, so that triage needs no drill-in.
18. As the operator, I want to open any run from the grid, so that the two screens form one workflow.
19. As the operator, I want to select a node and see its owner, kind and attempt number, so that retried nodes are legible as retries.
20. As the operator, I want to see each gate's individual result, so that I know which check failed rather than only that gates failed.
21. As the operator, I want to read the exact prompt an agent was given, so that I can debug a bad outcome without transcript archaeology.
22. As the operator, I want the repair prompt persisted too, so that I can see the diff and failure text the repair agent actually reacted to.
23. As the operator, I want all three prompts of a feat run retained, so that plan, build and test stages are each auditable.
24. As the operator, I want each role's resolved model shown on its rail, so that I can tell what actually ran rather than what was requested.
25. As the operator, I want the agent's tool list and MCP servers recorded, so that a behavioural anomaly can be checked against its configuration.
26. As the operator, I want the system prompt to have a visible slot, so that its current absence is a stated fact rather than an omission.
27. As the operator, I want Codex runs to render structurally alongside Claude runs, so that provider choice does not create a blind spot.
28. As the operator, I want Codex config to say "not captured on this provider" rather than render empty, so that absence of capture is never mistaken for absence of configuration.
29. As the operator, I want the view to update without refreshing, so that watching a run is passive.
30. As the operator, I want to launch it with one command, so that watching a run costs nothing to start.
31. As the operator, I want it bound to loopback with a token, so that no other local process can read every prompt the factory has compiled.
32. As the operator, I want the view to be strictly read-only, so that it can never become a way to break a run.
33. As the operator, I want a truncated or in-flight journal to render up to its last complete line, so that a live run is watchable and a crashed one is still readable.
34. As the operator, I want a run missing its journal to be skipped with a notice rather than crash the view, so that one bad artifact never costs me the others.
35. As the builder agent, I want the factory to persist prompts and config independently of any view, so that the archive survives the FE being rewritten or abandoned.
36. As the builder agent, I want the projection to be a pure function, so that every rendering rule is testable without a browser or a server.
37. As the operator, I want a heartbeat that also makes `just status` honest, so that the work pays off even outside the view.

## Implementation Decisions

1. **Six seams total, four of them already exist.** Reused: the injected journal sink (new event types ride it), the timer seam (the heartbeat chain), the agent query seam (prompt and config capture), and the existing journal reader (the view's read model builds on it — **no second parser**). New: the pure projection, and the server edge.
2. **The pure/edge split is mandatory, not stylistic.** The projection performs no I/O and holds every rendering rule; the server is a thin shell with injected dependencies. This is the Art. IX shape used throughout the codebase, and it is what makes the feature testable with plain arrays.
3. **The browser is never a test seam.** No headless browser, no DOM assertions, no new test tier. If a behaviour cannot be asserted at the projection, it is presentation and this spec does not specify it.
4. **The journal remains the sole run history** (Art. VIII). Sidecars are addressed *from* the journal; nothing is discoverable only by directory scan.
5. **Prompt sidecars are content-hashed and length-recorded**, so a truncated or externally edited body is detectable rather than silently rendered.
6. **Persistence never blocks a run.** A failed sidecar write is journaled and the run proceeds — the same tolerance stance tracing and capture already take. A sick archive must never convert a green run into a blocked one.
7. **Heartbeat cadence 15 s, staleness 60 s**, both named constants carrying their rationale. ~4 appends per minute, negligible against 2 000+ capture records per run.
8. **Dots bind to lanes by time window, never by session id** — session id is 1:N across resumed sessions.
9. **`target` is added to the existing run-start event**, not to a new event.
10. **Existing journal consumers must not regress.** New event types are additive; the status reader and PR-body statistics keep working against journals that predate them, and against journals that contain them.
11. **Zero new runtime dependencies.** `Bun.serve` and SSE ship with the runtime.
12. **Read-only by construction**: the server exposes no mutating route at all. This is not a permission check to be misconfigured — the capability is absent.
13. **Loopback bind plus a token**, printed to the terminal — per-launch by default, operator-fixable via `ADW_WEB_TOKEN` (and `ADW_WEB_PORT`; `--port` wins).
14. **Shipped as an `adw web` subcommand** with a `just watch` recipe, consistent with every other operator surface. This widens the tested CLI surface and that suite must be updated.
15. **No cLens deep-link in v1.** It would require a change in the cLens repo plus a launch-mode dependency, to save a manual step. Deferred, not rejected.
16. **The view reads capture JSONL directly** from the run directory. These are the factory's own files in its own directories, in a flat four-key shape — this is not reaching into another tool's internals.
17. **The system prompt has a UI slot but no value decision here.** Today's honest render is "none (empty override)"; choosing otherwise is `adw-sysprompt-01`.
18. **Codex renders structurally**, with configuration explicitly labelled uncaptured. There is **no Codex run on disk anywhere**, so nothing about that path can be verified against real data, and this spec does not pretend otherwise.

## Testing Decisions

**What makes a good test here:** it asserts external behaviour — given these journal and capture records, the projection yields this view — and never reaches into how the projection is structured. Every rule in §D is expressible as an array of records in, a view out. Prior art is `test/observability/journal.test.ts` (records in, parsed output out) and `test/pipeline/engine.test.ts` (fake clock, fake journal sink, assert transitions).

**Projection (the bulk of the suite).** Lane assignment for each node kind; a node with no end still renders as in-progress; **a resumed-session tool call lands in the repair lane, not the builder lane** — the regression test for the 1:N hazard, and the single most valuable test in this spec; heartbeat fresh → `running`; heartbeat stale → `unknown`; no heartbeat at all (pre-amendment journal) → renders without crashing and without claiming liveness; run-end present → `finished` regardless of heartbeat; missing capture → "no capture", distinct from captured-with-zero-calls; a watchdog trip stalls one lane and leaves the run healthy; a truncated final line is ignored and everything before it renders.

**Heartbeat (engine).** With a fake timer and fake clock: beats appear at cadence; the chain cancels on run-end, on abort, and on a thrown node; the timer is unref'd; no beat is emitted after a terminal event.

**Persistence (query seam).** With a fake query seam: each of the four prompt sites writes its body and journals an index record naming path, length and hash; `system/init` fields survive to the journal; **a feat run yields three distinct prompts, not one** — the regression test for the overwritten-key defect; a failing sidecar write journals the failure and the run continues.

**Server edge (thin).** Byte-offset tailing emits only complete lines and resumes correctly across appends; a request without the token is refused; no mutating route exists.

**Unregressed.** The existing journal, status, engine and PR-body suites stay green with no assertion changes. Journals written before this amendment must still read.

## Out of Scope

- **Any control capability.** No stopping, pausing, retrying or dispatching from the view. Read-only was an explicit operator decision, and it is why no pid is recorded.
- **A scheduler, queue drainer, or watcher daemon** — still deferred by `adw-v1.md` §2, untouched here.
- **Detecting an alive-but-looping run.** Named as undetectable in v1 rather than silently unhandled.
- **A cLens deep-link**, and any embedded session/event inspector. Session interiors stay cLens's job.
- **Authoring or choosing a system prompt** — `adw-sysprompt-01`.
- **Persisting the Codex argv.** It is the richest configuration record on that path, but with no Codex run on disk it cannot be live-verified, and an unverifiable requirement is dead weight in a TDD ticket.
- **Cost display.** Ruled out on evidence: under the operator's subscription economics, marginal dollar cost is ≈ 0 and the scarce resource is rate-limit headroom. A dollar figure here would be a borrowed metric optimising the wrong variable.
- **Redaction or scrubbing of persisted prompts.** The 0700, gitignored, no-token-in-artifact posture is inherited wholesale. A redactor that silently mangles a prompt destroys the archive's value for the exact debugging case that justifies it.
- **Historical backfill.** Runs predating this amendment render with the structure they have and say what is missing; nothing is reconstructed or inferred.
- **Multi-user, remote, or authenticated-beyond-loopback access.**

## Acceptance

Staged. Each stage is independently valuable and independently shippable.

**Read the gate column before marking anything done.** Stages 1, 2, 4 and 5
require a **real dispatched run** — they are operator-executed bars, not
offline suites, and the repo has a documented failure mode here: `adw-m5-06`
is `done` with its billed live bar still owed and **untracked**, recorded only
in the ticket's own run log. No stage below may be checked off on offline
evidence alone, and any stage whose live bar is outstanding when the code
lands must be **carved into its own ticket**, not left implied.

| Stage | Gate |
|---|---|
| 1 Persistence | **operator — live run** |
| 2 Heartbeat | **operator — live run + a deliberate kill** |
| 3 Projection | offline suite |
| 4 The view | **operator — live run** |
| 5 Honest render | **operator — live run + a deliberate kill** |

1. **Persistence** — a real run writes all four prompts, the `system/init` config reaches the journal, and a feat run yields three distinct prompt bodies. *Verifiable with no FE in existence.*
2. **Heartbeat** — a real run emits beats at cadence and stops cleanly at every terminal path; a deliberately hard-killed run leaves a journal whose last beat is older than the staleness bound. *Also verifiable with no FE.*
3. **Projection** — the full suite green against fixtures drawn from real banked journals, not hand-written ones only.
4. **The view** — `adw web` serves both screens over the 55 banked runs, and a live run updates in place without a refresh.
5. **The honest-render bar** — with a run deliberately killed mid-build, the grid shows `unknown` within 60 seconds. **This is the acceptance test for the whole amendment**: it is the failure the operator has already hit by hand, and the reason any of this is worth building.

## Residual Risks

- **The grid has at most one live cell until `adw-par-01` lands.** Accepted knowingly; the grid is useful as a history browser regardless, and lifting serialization to feed a dashboard would be letting the view drive the factory's risk posture.
- **Heartbeat proves the process, not progress.** Stated in the view, not papered over.
- **Capture coverage is imperfect and getting worse** — **18 of 69** captured sessions carry no prompt (it was 4 of 53 a day earlier, so this is a widening gap worth its own investigation, not a fixed rate). Session-lifecycle events fire unreliably: `SessionEnd` **0** times across 69 sessions, `SessionStart` **3**. The view must therefore be built on tool-call events and the journal, never on lifecycle events.
- **Persisted prompts grow the run directory.** ~16 KB median × three per feat run is small against the workspaces already there, but `runs/` has no retention policy and this adds to an unbounded directory. Not solved here; named.
- **The Codex path is specified but unverifiable** until a Codex run exists on disk.
