---
id: adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live
type: bug
status: done
priority: 1
created: 2026-09-16
depends: []
attempts: [{"runId":"adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789515578821","branch":"adw/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789515578821/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789595055605","branch":"adw/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789595055605/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789600226674","branch":"adw/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-3","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789600226674/workspace","outcome":"blocked","provider":"claude","model":"sonnet"},{"runId":"adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789603151034","branch":"adw/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-4","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-fe-19-instant-nodes-stack-into-unreadable-boxes-and-the-drawer-goes-blank-while-live-1789603151034/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/63","provider":"claude","model":"sonnet"}]
---
# Zero-duration nodes pile into overlapping dashed boxes, and the drawer reads `—` for a node the Gantt is timing at 1m 14s

> Observed 2026-09-16 by the operator on
> `adw-auto-03-deterministic-baseline-1789515243292`, live, while `baseline`
> was still running. Two independent defects on the one screen whose entire
> job is "watch a run".

> **Amended 2026-09-17 — read `## Prior analysis` at the foot of this ticket
> FIRST.** Run `…-1789595055605` produced a complete plan, then lost its
> stream and was discarded. That plan is appended verbatim. It establishes
> that the line references in both `### Root cause` sections below have
> DRIFTED, and that the `MARKER_THRESHOLD_PCT = 7` collapse those sections
> describe as missing already exists. Trust the appended analysis over the
> root-cause prose where they disagree.

## Not a duplicate of `adw-fe-18`

`adw-fe-18-a-running-agent-node-sits-on-the-wrong-lane` (P1, `blocked`, 1
failed attempt) is about an **agent** node being drawn on the `code` lane and
mislabelled "deterministic · no tokens" while live.

Here the node is `baseline` — **genuinely deterministic**. The lane and the
"deterministic · no tokens" label are *correct*. These are different bugs in
the same view. Fixing `fe-18` will not fix either defect below.

---

## Defect A — consecutive instant nodes render as stacked boxes

### What is seen

Around the live `baseline` block: two to three nested dashed rectangles, inset
a few pixels from each other, no legible boundary between them. The operator's
words: *"this is not clear what is going on."*

### Root cause

`render-run.ts:621` sizes every node block by absolute percentage position,
with a floor:

```css
.nd{position:absolute;top:13px;bottom:13px;min-width:26px; … overflow:hidden;
    border:1px solid …; border-radius:8px}
.nd.k-det{background:transparent;border-style:dashed;
    border-color:var(--faint);border-left-color:var(--faint)}
```

A node that starts and ends inside the same millisecond gets `width: 0%` — but
`min-width: 26px` forces it to paint 26 px wide anyway, at its own `left%`.
Consecutive instant nodes therefore all land within a hair of the same `x` and
**paint on top of one another**.

`adw-fe-18`'s own measured table shows the shape exactly:

| node | left% | width% |
|---|---|---|
| baseline | 0.12 | **24.71** |
| baseline-green-check | 24.84 | **0.00** |
| assemble-plan | 24.84 | **0.00** |

Two zero-width nodes at an identical `left%`, each forced to 26 px. That is
the stack in the screenshot.

Because `.k-det` is `background: transparent`, they do not occlude each other
— the dashed borders show *through*, producing the nested-rectangle look
rather than one box hiding another.

Compounding it: a **live deterministic** node carries both `.k-det` and
`.live`. `.live` (`:658`) overrides only `border-color`, so the block keeps
`border-style: dashed` from `.k-det` and gains a blue dash plus the 30 px
`::after` shimmer (`:660`) — a third edge inside the same few pixels.

### What good looks like

- An instant node must be **visually distinct from a zero-length one** — a tick
  or marker on the lane, not a 26 px box pretending to have duration.
- Overlapping blocks must not be possible. Either collapse a run of instant
  nodes into one marker, or lay them out so their minimum widths cannot
  intersect.
- One block, one readable border. A live deterministic node should not stack
  dashed + live + shimmer edges within 3 px.

---

## Defect B — the drawer blanks every metric while the node is live

### What is seen

The drawer under the Gantt, for that same `baseline`:

```
DURATION  TURNS  BILLED  CONTEXT  S/TURN  TOK/S  TOK/S (MODEL)
   —        —      —        —       —      —         —

baseline
kind: in-progress
prompt not persisted
system prompt: not recorded
```

The Gantt, three centimetres above, is rendering **`1m 14s`** for the same
node.

### Root cause

`gantt.ts:74-82` — `usage` is

> Set only for agent-kind blocks, from `event.details.usage`; undefined for
> gate and deterministic blocks, **and for in-progress blocks (usage only
> lands on `node-end`)**.

and `metrics` is "undefined for any block with no `usage`". The drawer renders
`—` for each absent field. So a live deterministic node misses on **both**
counts at once.

### Why `—` is the wrong glyph twice over

1. **DURATION is knowable and already known.** The Gantt computes the live
   width from `start` + `now` (`render-run.ts:128`: *"an in-progress block's
   open end sizes against `now`"*). The drawer has the same block. It should
   read `1m 14s`, ticking.
2. **TURNS / BILLED / TOK/S will never exist for this node.** It is
   deterministic — the lane label says so, one panel over. `—` means "not
   known yet", which invites the operator to wait for a number that is never
   coming. "n/a", or simply not rendering those columns for a deterministic
   node, tells the truth.

### What good looks like

- **DURATION ticks live** for any in-progress block, matching the Gantt.
- Metrics that are *categorically inapplicable* (deterministic/gate nodes)
  render distinctly from metrics that are *pending* (agent node mid-flight).
  Three states, three glyphs — not one `—` for all of them.
- The operator's ask, verbatim: *"would love to get live data rendered in the
  below block."*

---

## Build protocol

TDD, Article I — no source before a reviewed, red test.

1. **Defect A**, against the pure projection/layout seam (`buildGanttView` and
   whatever computes `left%`/`width%`): given a journal with two consecutive
   zero-duration `node-start`/`node-end` pairs, assert the produced layout
   cannot place two blocks such that their rendered minimum widths overlap.
   This is a pure function of the journal — no DOM, per Art. III.
2. **Defect B**, against the drawer projection: given an in-progress block
   with no `usage`, assert a live duration is produced from `start` + an
   injected clock; and assert a deterministic block's inapplicable metrics are
   marked distinctly from an agent block's pending ones. Inject the clock — no
   real time, no snapshot flake.
3. Confirm red. Present for operator review.
4. Implement to green.
5. `bun run lint && bunx tsc --noEmit && bun test` all green.

## Coordination note — RESOLVED 2026-09-17

`adw-fe-17-the-chain-before-it-runs` has **merged**. The `pairs` conflict this
section originally warned about no longer exists; the appended plan was written
against the post-`fe-17` tree and confirms it. Nothing here blocks dispatch.

## Out of scope

- `fe-18`'s lane misattribution for live **agent** nodes (its own ticket).
- The journal-schema change that would carry `kind` on `node-start`
  (`gantt.ts:23-28` names it as the proper fix, out of scope there and here).
- `prompt not persisted` / `system prompt: not recorded` — separate capture
  concern, not a rendering defect.

---

## Prior analysis — salvaged plan from run `…-1789595055605`

> Verbatim from that run's `.adw/artifacts/plan.md`. The plan stage wrote this
> complete, then its stream stalled six minutes later and the engine discarded
> the run (three cold restarts, identical prompt, 36 min, no fix). The
> reasoning is sound and was verified against the current tree — it is here so
> this re-run does not pay for it a fourth time.
>
> It is prior analysis, **not** a spec. Re-derive anything you doubt; the
> `## Build protocol` above still binds, red tests first.

## Repro-strategy plan — adw-fe-19

Two independent defects on the run-console screen. Both are in
`src/web/render-run.ts`; neither touches `src/web/gantt.ts`'s pairing logic
or `src/web/drawer.ts`'s `buildDrawerView` (those are already correct for
this ticket's scope — see analysis below). Coordination note in the ticket
is now moot: `adw-fe-17-the-chain-before-it-runs` has already merged
(`a6dce83`) into this checkout, so there is no live `pairs` conflict to
re-check.

---

### Defect A — consecutive zero-duration blocks stack

#### Where the bug actually lives today

The narrow-block marker collapse (`MARKER_THRESHOLD_PCT = 7`,
`render-run.ts:36`) already exists in this checkout — it was not present
when the ticket's root-cause section was written, or the ticket's author
was looking at a slightly stale line reference (`render-run.ts:621`; the
`.nd` CSS block is at `render-run.ts:663-671` now). This checkout already
collapses a block narrower than 7% width to a single-glyph `.nd-marker`
button (`renderBlock`, `render-run.ts:432-437`).

**That collapse does not fix the bug** — it only changes what each
individual too-narrow block looks like. `renderLanes` (`render-run.ts:219-226`)
still calls `renderBlock` once per `GanttBlock`, independently, with no
awareness of siblings beyond `spillRoom`'s neighbour-distance check (which
only gates the *gate-block-exemption*, not general placement). Two blocks
with identical (or near-identical) `left%` — the exact shape produced by
two zero-duration `node-start`/`node-end` pairs at the same timestamp —
still render as two separate `.nd`/`.nd-marker` elements at the same `left`.
Both inherit `min-width:26px` from the base `.nd` rule (`.nd-marker` does
not override `min-width`), so both paint the same 26px box at the same
spot. The dashed, transparent-background `.k-det` border (`render-run.ts:670`)
means neither occludes the other — they show through as the "nested
rectangles" in the report.

Confirmed against the ticket's own measured table: `baseline-green-check`
and `assemble-plan` both land at `left:24.84% width:0.00%` — literally
identical position. Reproduced below with the exact same numbers.

#### The seam to test

`buildGanttView` (`src/web/gantt.ts`) → `renderLanes` (`src/web/render-run.ts`).
Both are pure: journal records in, typed `GanttView` out; `GanttView` +
`now` in, an HTML string out. No DOM needed — the existing test suite
(`test/web/render-run.test.ts`) already asserts on `renderLanes`'/`renderRunPage`'s
raw HTML output via regex extraction (see `blocksOf`, `test/web/render-run.test.ts:1208-1224`,
already used by the closely-related "narrow gate block" tests at
`test/web/render-run.test.ts:1237-1347`). Reuse that exact helper rather
than inventing a new one — it already parses `{classes, left, width, title}`
per rendered `.nd`/`.nd-marker` element from the HTML string.

#### Test to write

**File:** `test/web/render-run.test.ts` (co-locate with the existing
`blocksOf`-based tests; this is a `render-run.ts` rendering-layer bug, not
a `gantt.ts` pairing bug — `buildGanttView`'s output for this journal is
already correct: three finished blocks + one in-progress block, all in the
one `workspace` lane, per its own documented lane-assignment rule).

**New describe block:** `"consecutive zero-duration blocks do not stack (ticket adw-fe-19, defect A)"`.

**Setup — a hand-built journal, fed through the real `buildGanttView`
(imported from `../../src/web/gantt`), mirroring the ticket's own measured
numbers exactly:**

```
records = [
  { runId, ticketId, timestamp: 0,      event: { type: "node-start", node: "baseline" } },
  { runId, ticketId, timestamp: 24_710, event: { type: "node-end",   node: "baseline", outcome: "pass" } },
  { runId, ticketId, timestamp: 24_840, event: { type: "node-start", node: "baseline-green-check" } },
  { runId, ticketId, timestamp: 24_840, event: { type: "node-end",   node: "baseline-green-check", outcome: "pass" } },
  { runId, ticketId, timestamp: 24_840, event: { type: "node-start", node: "assemble-plan" } },
  { runId, ticketId, timestamp: 24_840, event: { type: "node-end",   node: "assemble-plan", outcome: "pass" } },
  { runId, ticketId, timestamp: 24_850, event: { type: "node-start", node: "build" } }, // trailing, in-progress — matches the ticket's "live" framing
]
```

None of these node-ends carry a `details.kind` of `"agent"`, so
`buildGanttView` correctly files all four blocks into the single
`WORKSPACE_LANE` ("workspace") — this matches the ticket's own "the lane
... is correct" framing; defect A is not a lane-assignment bug.

```
const view = buildGanttView(records);
const { html } = renderLanes(view, 100_000); // now = 100_000, matching the ticket's span
const blocks = blocksOf(html); // reuse the existing helper, render-run.test.ts:1208
```

With `now = 100_000`: `baseline` → `left:0% width:24.71%`; `baseline-green-check`
→ `left:24.84% width:0.00%`; `assemble-plan` → `left:24.84% width:0.00%`
(both collapse to `.nd-marker` under the existing `MARKER_THRESHOLD_PCT`
rule — irrelevant to the bug); `build` → `left:24.85% width:75.15%`,
in-progress. This is the ticket's exact measured shape, reproduced from
first principles rather than copied as a fixture.

**Assertion (fails today, passes once fixed):**

```
const lefts = blocks.map((b) => b.left);
expect(new Set(lefts).size).toBe(lefts.length);
```

Today: `lefts = [0, 24.84, 24.84, 24.85]` → `Set` size `3` ≠ length `4` →
**fails**, because `baseline-green-check` and `assemble-plan` render at the
identical `left`. This is the sharpest, implementation-agnostic way to
state "two blocks must not paint on top of each other": since every `.nd`/
`.nd-marker` box has a strictly positive `min-width` (`26px`, `render-run.ts:663`,
inherited by `.nd-marker` unmodified), two blocks at an identical `left%`
are guaranteed full pixel-for-pixel overlap regardless of the lane track's
actual rendered width — no assumed viewport width needed to make this
claim. It does not presuppose which of the two acceptable fixes the build
stage picks (collapsing the run of instant nodes into one marker — which
drops the duplicate `left` entirely — or spacing them apart in `left%` so
they no longer coincide) — either satisfies the assertion; it also holds
even if the build stage keeps `renderLanes`'s current one-block-at-a-time
signature and instead makes it aware of siblings within the same lane
(e.g. a small pass that nudges/merges runs of identically-positioned
blocks before rendering).

**Behavioral gate this test lives under:** the same file/describe grouping
as the existing "a gate block never paints its verdicts over the next
block" suite (`test/web/render-run.test.ts:1237`) — both are instances of
"a block must never visually collide with its neighbour on the same lane
track," just triggered by different geometries (spill vs. identical
position).

---

### Defect B — the drawer blanks every metric while the node is live

#### Where the bug actually lives today

`drawer.ts`'s `buildDrawerView` is **not** the source of this bug. Its
`kind` field already correctly reports `"in-progress"` for a live block
(`drawer.ts:100-102`) and `"deterministic"`/`"gates"`/`"baseline"` for a
finished non-agent one — that classification is accurate as far as it
goes, and this ticket does not need to change it.

The actual bug is entirely in `render-run.ts`'s `renderCard`
(`render-run.ts:533-588`), which renders the `card-stats` block (duration/
turns/billed/context/s-per-turn/tok-s/tok-s-model):

```
const m = block.metrics;
const t = m?.time;
const dur = m?.durationMs;
...
${stat("duration", fmtMs(dur))}${stat("turns", fmtNum(m?.turns))} ...
```

Two compounding problems, both visible here:

1. **`renderCard` has no `now` parameter at all** (`render-run.ts:533-538`;
   contrast `renderBlock`, which already takes `now`, `render-run.ts:396-402`,
   and is called with it from `renderLanes`). `dur` is sourced only from
   `block.metrics?.durationMs`, which — per `gantt.ts`'s own documented
   contract (`gantt.ts:74-76`) — is `undefined` for any in-progress block,
   agent or not, because `usage`/`metrics` only land on `node-end`. So
   `fmtMs(dur)` always renders `"—"` for a live block's DURATION, even
   though `renderBlock`, three lines up in the same render pass, computed
   the live block's width from `end ?? now` (`render-run.ts:404`) and is
   already painting a real, ticking duration on the Gantt bar itself. The
   fix's minimal footprint: thread `now` into `renderCard` (mirroring
   `renderBlock`'s existing signature) and compute
   `dur = block.inProgress ? now - block.start : m?.durationMs`.

2. **Every other stat's `"—"` fallback (`fmtNum`/`.toFixed() ?? "—"`) does
   not distinguish "will never exist" from "not in yet."** A finished
   `det`/`gate`-kind block (which never carries `metrics`, by construction
   — `kindOf`, `render-run.ts:95-99`) and a genuinely in-progress
   *agent* block mid-flight (metrics truly pending, will land at
   `node-end`) render byte-identical `"—"` today. The signal needed to
   separate them already exists and is already passed into `renderCard`:
   its own third parameter, `role` (`render-run.ts:536`). Every non-agent
   block — finished or in-progress — is filed into the single
   `WORKSPACE_LANE` ("workspace") by `buildGanttView`; every agent block
   (once finished) gets its own lane named after itself. `role ===
   WORKSPACE_LANE` is therefore already the exact "the lane label says so,
   one panel over" signal the ticket itself names — the same fact that
   already drives `renderLanes`'s `isWorkspace ? "deterministic · no
   tokens" : ...` label (`render-run.ts:233`). No new field needs to be
   threaded from `drawer.ts` or `gantt.ts` for this.

   **DURATION is exempt from this "n/a" treatment** — it is knowable for
   every kind (a deterministic node still takes wall-clock time). Only the
   six agent-specific stats (turns/billed/context/s-per-turn/tok-s/
   tok-s-model) are candidates for "n/a".

   Known, accepted scope boundary (shared with `adw-fe-18`, not this
   ticket's job): a live block's *true* future kind (agent vs.
   deterministic) is not yet knowable from the journal — `kindOf` already
   documents this ("correctly classifies it 'det' while running... the fix
   is a journal-schema change, out of scope," `render-run.ts:90-91`). So a
   genuinely-live *agent* node's stats would, under the `role ===
   WORKSPACE_LANE` rule, also read "n/a" rather than "pending" until it
   finishes and migrates to its own lane — that misclassification is
   `fe-18`'s territory, not introduced or fixed here.

#### The seam to test

`render-run.ts`'s `renderCard`, reached the same way the existing
`card-stats` tests already reach it (`test/web/render-run.test.ts:976-1010`,
the "tok/s (model)" describe block): via `page(...)` (the test file's own
`renderRunPage` wrapper, `test/web/render-run.test.ts:61-68`) on a
hand-built `GanttView`, asserting on substrings of the full returned HTML.
No DOM, no drawer/journal fixtures needed — `drawer` can stay `undefined`
(pass `new Map()`), since `card-stats` never reads it; only `block`,
`role`, and (after the fix) `now` feed it.

#### Tests to write

**File:** `test/web/render-run.test.ts`. **New describe block:**
`"the drawer stops blanking a live block's metrics (ticket adw-fe-19, defect B)"`.

**Test 1 — DURATION ticks live instead of reading `"—"`:**

```
const html = page(
  viewOf([
    lane("workspace", [
      block({
        node: "baseline",
        start: 0,
        end: undefined,
        inProgress: true,
        outcome: undefined,
      }),
    ]),
  ]),
  new Map(),
  74_000, // now — 1m 14s after start, the ticket's own measured figure
);
const durationStat = /<span>duration<\/span><b>([^<]*)<\/b>/.exec(html);
expect(durationStat?.[1]).toBe("1m 14s");
```

Today: `renderCard` never receives `now`, so `dur = block.metrics?.durationMs`
= `undefined` → `fmtMs(undefined)` = `"—"` → **fails** (`"—" !== "1m 14s"`).
Once `now` is threaded through and `dur` falls back to `now - block.start`
for an in-progress block, this reads `"1m 14s"` — the exact figure the
ticket says the Gantt bar already shows for this same block.

**Test 2 — a finished deterministic/gate block's agent-only stats read
`"n/a"`, not `"—"` (the general rule, reachable without liveness at all):**

```
const html = page(
  viewOf([
    lane("workspace", [
      block({ node: "baseline-green-check", start: 0, end: 100 }),
    ]),
  ]),
);
expect(html).toMatch(/<span>turns<\/span><b>n\/a<\/b>/);
expect(html).toMatch(/<span>billed<\/span><b>n\/a<\/b>/);
```

Today: `role === "workspace"` is not consulted anywhere in `renderCard`;
`fmtNum(m?.turns)` renders `"—"` unconditionally when `m` is `undefined` →
**fails**. After the fix, a `workspace`-lane block's agent-only stats read
`"n/a"` — true whether the block is finished or in-progress, since the
`role` check does not need to branch on `block.inProgress` (only DURATION
does).

**Test 3 — the exact reported scenario: ticking DURATION and `"n/a"`
agent-stats on the SAME live block, at once:**

```
const html = page(
  viewOf([
    lane("workspace", [
      block({
        node: "baseline",
        start: 0,
        end: undefined,
        inProgress: true,
        outcome: undefined,
      }),
    ]),
  ]),
  new Map(),
  74_000,
);
expect(html).toMatch(/<span>duration<\/span><b>1m 14s<\/b>/);
expect(html).toMatch(/<span>turns<\/span><b>n\/a<\/b>/);
```

Today: **fails on both** — `"—"` for duration, `"—"` for turns. This is
the single test that most directly reproduces the ticket's own screenshot
(the exact node name, the exact "1m 14s" the Gantt already shows, the
exact set of blanked stat rows).

**Suggested (non-blocking) companion — not part of the red set, since it
already passes today and stays passing; a regression guard the build
stage may add alongside:** a hand-built block on a non-`"workspace"` lane
with `inProgress: true` and no `metrics` should keep reading `"—"` (not
`"n/a"`) for its agent stats — pinning that the `role === WORKSPACE_LANE`
rule does not overreach onto a lane that might turn out to be a genuinely
pending agent block.

**Behavioral gate these tests live under:** alongside the existing
`"the dock card's tok/s (model) stat"` / `"the dock card's model/tool
split bar"` describe blocks (`test/web/render-run.test.ts:899-1010`) — all
are instances of "the per-block dock card renders `block.metrics`-derived
stats correctly," just exercising the previously-untested in-progress and
categorically-inapplicable branches.

---

### Implementation-adjacent notes for the build stage (not part of the red tests)

- `renderCard`'s signature needs a `now: number` parameter; `renderRunPage`
  (`render-run.ts:339-346`, already has `now` in scope) needs to pass it
  through at its one call site (`render-run.ts:350`).
- The "n/a" glyph is a literal choice matching the ticket's own suggested
  text ("`'n/a'`, or simply not rendering those columns"); nothing in this
  plan requires that exact string beyond what the tests above pin — but
  picking one consistent literal (rather than omitting columns) is the
  smaller diff against the existing fixed 7-row `card-stats` layout.
- No changes anticipated in `drawer.ts` or `gantt.ts` for either defect —
  both are `render-run.ts`-only, per the analysis above. If the build
  stage finds this plan's `role === WORKSPACE_LANE` signal insufficient
  once implementing, the fallback is threading `drawer?.kind` (already a
  `renderCard` parameter) instead — equivalent in every case this plan's
  tests exercise, since `WORKSPACE_LANE` and `drawer.kind !== "agent"`
  agree for every reachable `GanttBlock` today.
