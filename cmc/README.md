# CMC tickets (ADW factory)

Work orders for the ADW factory (`~/personal_project/adw-factory`, target
`cmc`, `ticketsDir: ~/adw/backlog/cmc`). They live in the central backlog, never
in the Go1 repository. It rewrites `status:` and
`attempts:` itself; never edit `attempts:` by hand.

## Content quality page, release 1 (`cqc-fe-*`)

**Base branch: `cqc/release-1`** (operator decision 2026-09-28). The CLB team owns the CMC and
reviews from another timezone, so the page is built on a feature branch: `targets/cmc.json`
has `"base": "cqc/release-1"`, every factory PR targets it, and the operator merges there
(the branch has no protection or ruleset). The finished page goes to `master` as one PR for CLB.
Keep the branch current with `master` before that PR. 01 (#135) and 02 (#136) are merged
there; the combined branch was verified green (lint, 601 tests, build) on 2026-09-28.

Source spec: `docs/cqc/spec-cqc-fe-release-1.md`. The API contract is in
`docs/cqc/backend-contract.md` and the decisions in `docs/cqc/decisions-2026-09-26.md`.

Dependency graph (`depends:` is enforced; a blocker must be `done`, that is merged):

```
01 drawer-keeps-keyboard-focus ─────────────┐
02 gated-content-quality-page ── 03 list ───┼── 06 drawer-checks ── 07 drawer-navigation ── 09 technical-tab
                                  │  │  │    │        │  │  │
                                  │  │  └────┼────────┘  │  └── 08 evidence-tab
                                  │  04 row-facts ───────┼── 10 trigger-checks
                                  │          └───────────┴── 11 live-refresh
                                  05 filters ────────────────── 12 resolve-validate (also needs 06)
```

Frontier at start: 01 and 02. **01-12 are all merged into `cqc/release-1`** as of 2026-09-29.

## After the first browser, 2026-09-29

The page was mounted locally against the deployed CQC dev API for the first time.
Evidence: adw-factory `ai_docs/2026-09-29-cqc-dev-integration-findings.md`.

```
13 dev-server-type-error         (bug,   done)  PR #147 - npm start did not compile
15 the-page-runs-locally-...     (chore, queued) the same gap, closed properly + 2 more
14 one-contract-types-from-openapi (feat, queued) BLOCKED on cqc-be-14 (other repo)
```

`16` (manual) opens the single PR to `master` for CLB — and carries the decision that has to
be made first: `cqc-fe-10` and `cqc-fe-12` shipped Check, Run-again, Resolve, Validate and
Reopen against endpoints that land in backend release 2 and 3. All four return a raw API
Gateway 403 today. Ship read-only behind a config flag, or hold the PR. Product call.

**There is no other CMC release-2 work.** The trigger UI, the polling and the in-progress
strip are already built and waiting on the backend.

## UI/UX pass, 2026-09-30

Audit: adw-factory `ai_docs/2026-09-30-cqc-frontend-ux-audit.md`. Token discipline in this
feature is clean - no hex, no rgb, no px spacing. The problem is composition: the page states
everything at equal weight, repeats the same six checks up to three times, and is ~1848px
wide against a 1440px laptop.

```
17 four-things-that-render-broken  (bug,   P1) tabs stack, chips overlap, buttons stretch, page overflows
18 the-row-leads-with-a-verdict    (feat,  P1) 7 cols -> 5, <=1180px, rows 240px -> 96px   [needs 17]
19 the-drawer-says-each-check-once (feat,  P2) three restatements -> one; one fact block   [needs 17]
20 the-page-stops-shouting...      (chore, P2) premature validation, ragged filter row, trigger placement
```

`17` and `20` can run now. Three of `17`'s four defects share one cause: Go1d `View` is a
flex column with `align-items: stretch`, and this code does not counter it.

Design direction, for anyone picking these up: the signature is the **journey strip** -
six checks, always the same order, `launch -> navigate -> complete -> not-forced -> media ->
resume`. It is the one ordered device that is honest here, because the checks really are a
sequence. Everything else gets quieter.

`15` is independent and can run now. **`14` must not be dispatched until `cqc-be-14` is
merged** - it rewrites `backend/openapi.yaml` under amendment BE-38, and generating against
today's file would bake in the shape being replaced. That blocker is *not* enforceable in
`depends:`: the guard only resolves ids inside this target's own backlog and silently treats
a cross-target id as met. The ticket's own heading and the operator are the guard.

## Design pass, 2026-10-01

The operator compared the build with the approved prototype (variant B + drawer 1) and found
it "clearly not close". Full review, coverage matrix and evidence: adw-factory
`ai_docs/2026-10-01-cqc-fe-design-pass/TASKLIST.md` (217 headless captures of the app and the
prototype, DOM collision/overflow/focus probes, re-runnable sweep scripts).

**Why it drifted:** `targets/cmc.setup.sh` excludes `docs/cqc/prototype/` from every worktree
(partner data in `data.js`), so no build agent ever saw the prototype. These tickets carry its
structure in writing, translated to Go1d.

```
27 go1d-containers-stop-breaking-the-layout (bug,  P1) pills, stretched/overflowing buttons, bullets, mono, focus
28 the-drawer-header-fits-in-a-quarter      (feat, P1) 475px header -> ~215, verdict vocabulary   [needs 27]
29 the-strip-and-tabs-stay-put-...cards     (feat, P1) pinned 6-cell strip, underlined tabs, check cards [needs 28]
30 evidence-is-a-list-beside-its-viewer     (bug,  P1) overlapping rows, two-pane Evidence tab     [needs 29]
31 the-row-is-the-click-target              (feat, P1) whole-row target, page fits, 9 columns     [needs 27]
32 the-top-of-the-page-is-one-calm-band     (feat, P2) paste bar on top, captions, one filter row [needs 31]
33 actions-look-like-actions-...quietly     (bug,  P2) config-down once, footer buttons, radios   [needs 28, 32]
```

Two lanes after 27: drawer (28 → 29 → 30) and list (31 → 32). 33 joins them. 31 and 32
deliberately reverse fe-18's five columns and fe-20's trigger-at-the-bottom, to match the
prototype. Backend half: `cqc-be-46` (gateway-error + bucket CORS) in the `cqc` backlog;
`cqc-be-41` (handler CORS) is already in progress.

## Design review, 2026-10-02

The operator found the drawer "super messy": unstructured data, no clear separation between
tabs, compacted text, and the snapshot viewer squeezed at the top of a mostly empty screen. Full
review (82 items, diagnosis, wireframe): CMC `docs/cqc/design-review-2026-10-02.md` (overlaid in
every worktree). Operator decisions: **D-R1** the run report becomes a page `/quality/:loId`
and the drawer a peek; **D-R2** every N/A stays visible, rendered quiet (subtle, `fontSize={1}`).

```
34 nothing-renders-at-zero-px          (bug,    P1) Go1d fontSize={0} is 0 px: summary labels, strip states
35 the-go1d-primary-button-is-readable  (manual, P2) 2.3:1 contrast, Go1d accent, design-system call
36 each-count-says-what-it-counts       (chore,  P2) "Snapshot count 36" vs "Snapshot 9 of 52"
37 the-run-report-is-a-page             (feat,   P1) /quality/:loId + peek drawer           [needs 34]
38 the-journey-track-is-the-navigation  (feat,   P1) signature: 6 numbered stations         [needs 37]
39 the-header-says-the-verdict...       (feat,   P1) 270px header -> one verdict sentence   [needs 37]
40 checks-come-before-limitations...    (feat,   P1) order + plain words for field names    [needs 38]
41 the-evidence-viewer-fills-the-height (feat,   P1) no 320px caps, one scroll region       [needs 37]
42 the-scorm-trace-is-readable          (feat,   P2) +0.000s, writes-only, no N/A for n/a   [needs 41]
43 snapshots-are-a-tree-...             (feat,   P2) mono collapsible tree, step pairing    [needs 41]
44 technical-values-are-mono-...        (chore,  P3) hashes, chips, tabular figures         [needs 36]
45 the-list-fits-its-data               (feat,   P2) one width, stat band filters, headers
46 one-status-vocabulary-and-quiet-na   (chore,  P2) one vocabulary, quiet N/A renderer     [needs 38 39 40 44 45]
```

Frontier now: 34, 36, 45 (and 35 for the operator). Lanes after 37: header (39), track then
checks (38 -> 40), evidence (41 -> 42, 43). 46 sweeps last.

## Before the first dispatch (operator)

1. Land `docs/cqc/` (without `docs/cqc/prototype/`), `.agents/skills/cqc-frontend/`,
   `.agents/design-system/` and the `AGENTS.md` change on `master`. The factory
   cuts every worktree from `origin/master`, so the agent cannot see untracked files.
2. ~~Add the Statsig legacy JS SDK before `cqc-fe-02`.~~ Dropped 2026-09-28: the
   CMC ships internal features without a Statsig gate (spec D-1 amended).
3. `adw-store-01-tickets-dir` must be merged in adw-factory, and `targets/cmc.json`
   must set `"ticketsDir": "~/adw/backlog/cmc"`. Until then the factory reads
   `<repo>/tickets` only.

## Running

```bash
cd ~/personal_project/adw-factory
TARGET=cmc just next                          # what is runnable now
TARGET=cmc just run-codex cqc-fe-01-drawer-keeps-keyboard-focus
```

Every ticket pins `model: gpt-5.6-sol`, run through the Codex CLI's ChatGPT
login (the Go1 subscription in `~/.codex`). Gates (tslint, jest, build; the build type-checks)
run in Docker `node:14.21.3`.
