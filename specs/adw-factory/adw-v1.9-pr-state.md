# Amendment v1.9 — PR state on the card: did CI pass, and did a human validate it

> **Status:** proposed 2026-09-17, from the investigation in
> `ai_docs/2026-09-17-github-ci-review-on-card.md` (every figure below is
> measured there, on this machine, on that date).
> **Amends:** `adw-v1.2-live-view.md` — Decision 11 (*"zero new runtime
> dependencies"*, now bounded rather than absolute) and the board card's
> content contract. Also extends the `TargetConfig` schema from
> `adw-v1-plan.md` §4.
> **Binding:** `constitution.md` — unchanged. Art. IV (humans at the two ends)
> is *strengthened* here: the board finally shows what the human did.
> **Implemented by:** three tickets, staged in §Acceptance.

## Problem Statement

**The card's last word is "the PR opened."** `renderTicketCard` shows the lane,
the mini-Gantt, `green`/`blocked`, cost, wall time and tokens — every one of
them a fact about the *factory*. Not one is a fact about the *artifact the
factory produced*. `grep '\bpr\b' src/web/*.ts` finds only comments naming
`open-pr` as a node.

So the board answers "did the machine finish" and goes silent on the two
questions that actually decide whether the work is done:

1. **Did remote CI pass?** Local gates went green — that is what unlocked
   `push`. Remote CI is stricter (different OS, cold cache, no warm
   `node_modules`) and it is checked *nowhere the operator can see*. Today
   `ci-round` reads it, acts on it, and the board never learns the answer.
2. **Did a human validate it?** On `adw-factory` and `clens` the answer is
   "you merged it". On the **7 Coorpacademy targets** — `api-content`,
   `bricklane-persona-hooks`, `coorpacademy`, `coorpacademy-lambda`,
   `coorpacademy-oplog`, `serverless-plugins`,
   `translated-language-service` — review is mandatory and a reviewer may have
   requested changes hours ago. The board would still show a serene green
   `success`.

**A green card can be a lie.** A run that ends `green` and is then closed by a
reviewer renders identically to one that was merged. The factory's terminal
outcome is about the *run*; the PR's fate is about the *work*, and the card
conflates them.

The operator's workaround today is to leave the board, open GitHub, and check
by hand — once per card. That is the exact "read journals one at a time"
problem v1.2 §User-Story-11 was written to kill, displaced one step downstream.

## Solution

Two chips on each card, fed by **one `gh pr list` per target behind a TTL
cache**, read-only, and degrading to *nothing* whenever the answer is not
honestly known.

```
┌─────────────────────────────────────────┐
│ adw-fe-19-instant-nodes-stack…   12m ago│
│ bug · red test first, then the fix      │
│ ▁▃▅▂ mini-Gantt ▁▂▅▃                    │
│ ✓ success  ●●●●●●                       │
│ ◉ CI green   ⧗ awaiting review   #61    │  ← new row
│ $ ~2.98   ◷ 38m 30s   ◈ 4.58M           │
└─────────────────────────────────────────┘
```

Two **independent** axes, because they answer different questions and either
may legitimately be empty:

| Axis | Source | Values |
|---|---|---|
| **CI** | `statusCheckRollup` | green · red · cancelled · pending · *(no checks)* — only green and red carry colour |
| **Validation** | `state` + `reviewDecision` | merged · rejected · approved · changes requested · review required · awaiting |

The join from card to PR needs no new run data: branches are deterministic
(`<branchPrefix><ticketId>`, `-<n>` on a retry attempt) and `gh pr list`
returns `headRefName`. **This works retroactively on every banked run** — no
backfill, no migration, nothing reconstructed.

## User Stories

1. As an operator, I want a green/red CI dot on each card, so that I learn
   remote CI failed without opening GitHub.
2. As an operator, I want the failing check's name and a link to its job, so
   that I can go straight to the log instead of hunting through Actions.
3. As an operator, I want a pending CI state distinct from a passing one, so
   that I never read "not yet run" as "passed".
4. As an operator, I want to see that a PR was **merged**, so that the board
   tells me which work actually landed.
5. As an operator, I want to see that a PR was **closed**, so that a rejected
   piece of work stops reading as a success.
6. As an operator, I want to see **changes requested** on a team repo, so that
   I know a reviewer is waiting on me rather than the reverse.
7. As an operator, I want an approved-but-unmerged PR called out, so that I can
   merge the work that is already cleared.
8. As an operator, I want a card with no PR to say nothing at all, so that a
   blocked or still-running run never shows a misleading empty verdict.
9. As an operator, I want a solo repo's empty `reviewDecision` to render
   nothing, so that the absence of reviewers is not dressed up as a pending
   state.
10. As an operator, I want the chips to refresh while I watch, so that a CI run
    finishing is visible without a page reload.
11. As an operator, I want the board to work unchanged with `gh` broken,
    logged out, or offline, so that a network problem never costs me the run
    history I already had.
12. As an operator, I want the board never to exhaust my GitHub rate limit, so
    that leaving it open all day does not break `adw run`.

## Implementation Decisions

1. **A new optional `github: "owner/name"` field on `TargetConfig.`** Joins
   `sandbox`, `provider` and `systemPrompt` as an optional, run-scoped field
   in `targets/loader.ts`. **Required**, because `gh` resolves a repo from the
   cwd's git remote and **none of the 7 Coorp targets is cloned on this
   machine** — `targets/*.json` stores `repo` as a local path, which is not a
   GitHub identity. Absent → fall back to `git -C <repo> remote get-url
   origin`; both absent/failing → that target's cards carry no chips and the
   board says so in a notice. The 4 cloned targets need no edit.
2. **One `gh pr list` per target, never one per card, and always scoped to
   factory branches.** The query is
   `gh pr list --repo <owner/name> --state all --limit 300
   --search "head:<branchPrefix>" --json
   number,state,headRefName,statusCheckRollup,reviewDecision`.
   Per-card `gh pr view` at the SSE cadence would be 36 000 calls/hour against
   a **5 000/hour** budget — dead in 8 minutes.
3. **The `--search "head:<branchPrefix>"` scoping is mandatory, for
   correctness AND cost.** Measured against `CoorpAcademy/coorpacademy`
   (**13 359 PRs**): unscoped `--limit 300` with `statusCheckRollup` took
   **17.1 s / 180 KB**; scoped to `head:adw/`, **0.7 s**. `statusCheckRollup`
   is a nested per-PR fetch, so cost scales with rows returned. Unscoped
   coverage is also simply *wrong* — the 100 newest of 13 359 is about a week
   of traffic there, so a factory PR left unreviewed for a fortnight drops out
   of the window and its chip silently blanks. A search-hostile `branchPrefix`
   must **fail loudly**, never widen to every branch in the repo.
4. **The window is bounded at 300 and the bound is named.** Rows == limit →
   a notice. This repo carries 68 factory PRs growing ~10/day (peak 24), so it
   crosses 100 within days. The bound is self-limiting — it grows only as fast
   as the factory runs — and the escape hatch, when 300 stops sufficing, is to
   query exactly the ticket ids `runs/` holds (18 today) rather than a time
   window. The 17 s worst case rides the background TTL timer, never a
   request.
5. **TTL 60 s, on its own clock, decoupled from `SSE_POLL_MS`.** The SSE tick
   *reads* the cache and never triggers a fetch. 11 targets × 60 s = **660
   calls/hour, 13 % of budget**. Both constants carry their rationale in
   source, as `HEARTBEAT_STALE_MS` already does.
6. **The chip goes live over SSE with no new event plumbing.** `server.ts`
   already diffs the rendered fragment per connection and pushes only on
   change. A TTL refresh changes the chip's HTML, the diff fires, the push
   happens — the same mechanism that flips a liveness badge after 60 s of
   silence.
7. **One pure classifier, two input shapes.** `statusCheckRollup` exposes
   `status` + `conclusion`; `gh pr checks` exposes `state`. Feeding the rollup
   to today's `parseChecks` classifies **everything as passing** (no `state`
   field → no match → falls through). The rule is extracted into one pure
   function both shapes normalize into, and `ci-round.ts::parseChecks` is
   refactored onto it. Art. VIII: the board and the CI-repair round must never
   disagree about whether a PR is red.
8. **The branch→ticket join is longest-prefix against the board's known ticket
   ids — never a regex suffix strip.** `nextAttemptBranch` appends `-2`, `-3`
   … on retries, and a ticket id that legitimately ends in `-2` would
   otherwise bind to the wrong card, silently and unfalsifiably.
9. **The two axes are independent and separately absent.** `reviewDecision`
   empty renders **nothing** — not "unknown", not a spinner, not a
   placeholder. This is `card.ts`'s existing `captured` vs `noCapture`
   discipline: an honest absence is distinct from an absent-because-broken.
10. **Four distinct "no chip" causes, each with its own render.** (a) the run
   never reached `open-pr` — no PR exists; (b) the run has no recorded
   `target`, so no repo resolves (measured: **0 of 18** banked runs, but
   `render.ts` groups by `target ?? "unrecorded"`, so the path is live); (c)
   `gh` failed or is unauthenticated; (d) not fetched yet. Only (c) and (d)
   are transient, and only (c) earns a notice.
11. **The chip belongs to `latest`'s PR.** A card is a ticket, not a run
   (`TicketCard { ticketId, latest, attempts }`). "Older attempt merged, newer
   attempt open" is a real state; the card shows the newest attempt's PR, and
   the attempt dots keep their existing meaning.
12. **Read-only survives intact.** A `gh pr list` is a read. `sync-pr-state.ts`
    already carries the negative-capability guard proving no mutating PR
    command is constructed anywhere in `src`; that guard is **extended** to
    cover the new module. The `server.ts` grep test (no
    `"POST"`/`"PUT"`/`"DELETE"`/`"PATCH"` literal) is untouched by this work.
13. **Zero new runtime dependencies** — v1.2 Decision 11 holds in the sense
    that matters. `gh` is already an install prereq and already has four call
    sites. What changes is that the **web view** now has an *optional*
    network dependency, where before it had none. That is the real amendment,
    and §Decision 8's degradation rule is its price.
14. **The gh edge is injected, mirroring `GhRunner`.** `(args) => stdout`,
    bound by the CLI over `liveGh`, a scripted fake in tests. No test reaches
    the network; `cli.ts::interpretGh`'s existing exit-tolerance is reused
    rather than re-derived.
15. **The review axis is MEASURED, not assumed** (revised 2026-09-17 — an
    earlier draft of this decision claimed it was unverifiable; that was
    wrong, and the correction is the most useful thing in this spec). The
    operator's token reaches the Coorp org without a clone. Real data:

    | repo | APPROVED | REVIEW_REQUIRED | CHANGES_REQUESTED | empty |
    |---|---|---|---|---|
    | `coorpacademy` | 69 | 31 | 0 | 0 |
    | `api-content` | 74 | 23 | 1 | 2 |
    | `translated-language-service` | 79 | 21 | 0 | 0 |
    | `serverless-plugins` | 4 | 0 | 1 | **95** |

    Three consequences the design must carry:
    - **`reviewDecision` is the dominant signal on team repos** — 70–80 %
      `APPROVED`. It is not an edge case to be tolerated; it is the point.
    - **Empty is normal even inside Coorp.** `serverless-plugins` is 95 %
      empty — a low-traffic repo behaves exactly like a solo one. The
      render-nothing rule is load-bearing everywhere, not a solo-repo
      concession.
    - **`state` and `reviewDecision` disagree in production.** 13 of 88
      *merged* `api-content` PRs read `REVIEW_REQUIRED` — admin override, or
      a review dismissed by a later push. Decision 9's "two independent axes"
      is therefore measured fact, not a stylistic preference: a single merged
      "reviewed ✓" chip would be **wrong 15 % of the time on a real repo**.

## Testing Decisions

**What makes a good test here:** the same stance as v1.2 — records and
payloads in, view out, with no network and no browser. Every rule in
§Implementation-Decisions 5–9 is expressible as a pure function call.

**Classification (pure, the bulk).** A rollup of all-SUCCESS → green; any
FAILURE/ERROR → red; any QUEUED/IN_PROGRESS with no failure → pending; an
empty rollup → *no checks*, distinct from green; SKIPPED/NEUTRAL treated as
non-failing and documented as such; **the same fixtures fed through the
`gh pr checks` shape produce identical verdicts** — the regression test for
Decision 5, and the single most valuable test in this spec.

**The join (pure).** `adw/<id>` → that ticket; `adw/<id>-2` → that ticket;
a ticket id ending in `-2` against a branch for a *different* ticket → no
false bind (the Decision 6 trap); an unknown branch → no card; two PRs on the
same ticket → the newest number wins.

**Repo identity (pure).** Explicit `github` field wins; absent → derived from
a remote URL (both SSH and HTTPS forms); unparseable/absent → `undefined`,
never a guessed identity.

**The cache edge (fake clock, fake runner).** A second read inside the TTL
issues no second call; a read past the TTL issues exactly one; a throwing
runner yields an empty index **and a notice**, never a partial one that reads
as "this repo has no PRs"; N targets issue N calls, not N×cards.

**Rendering.** A card with no PR renders no chip row at all; `reviewDecision:
""` renders the state chip and no verdict chip; a red CI chip carries the
failing check's name and link.

**Unregressed.** `ci-round`'s own suite stays green through the `parseChecks`
refactor with no assertion changes. Every existing board/card/server test is
untouched — a target with no `github` field and no clone behaves exactly as
today.

## Out of Scope

- **Any mutation.** No approving, merging, closing, re-running CI or
  commenting from the board. Read-only was an explicit operator decision in
  v1.2 and is not reopened here.
- **Review feedback *text* on the card.** `CHANGES_REQUESTED` shows as a
  verdict; the bodies in `latestReviews[].body` need a second, per-PR
  `gh pr view` with its own cache. Deferred to a follow-up, not rejected —
  it is the obvious next increment once the Coorp targets are live.
- **The run screen and the v1.7 run panel.** Board card only in v1.9.
- **Journaling the PR URL on `open-pr`.** It is the Art. VI-correct home for
  it (`open-pr.ts:709` patches `ctx.data`, which dies with the process) and
  worth its own small ticket — but Decision 6's branch join resolves every
  banked run without it, so it is not on this critical path.
- **Non-GitHub CI providers.** All 11 targets are `github-actions`.
  `target.ci.provider` already exists as the seam if that ever changes.
- **Historical backfill or reconstruction.** Nothing is inferred; a run whose
  PR cannot be resolved says nothing.
- **Per-check detail on the card face.** The failing check's name goes in the
  chip's tooltip; a full check list belongs to a drawer that is out of scope.

## Acceptance

Staged. Stage 1 is verifiable with no UI in existence, matching v1.2's own
staging precedent.

**Stage 1 — the pure core + the target contract.** `github` field on
`TargetConfig` with validation; the repo-identity resolver; the shared
check classifier with `parseChecks` refactored onto it; the branch→ticket
join. No I/O, no UI. *Done when:* the pure suites are green and `ci-round`'s
existing suite is green with no assertion changes.

**Stage 2 — the gh edge. This is the risk stage.** The injected PR-index
runner, the TTL cache, the per-target fan-out, and the degradation-to-notice
path are routine. **`runWeb` gaining target awareness is not.** `runWeb` takes
`{ runsRoot }` and is deliberately target-agnostic; every *other* `adw` command
resolves exactly **one** target via `--target`, but the board renders **all**
targets at once, so it needs `targetsDir` plus a load-every-config path that
exists nowhere in `cli.ts` today. Expect this to surface a spec gap at pickup —
in particular, what the board does with a `runs/` directory whose recorded
`target` has no config file at all. *Done when:* a live `adw web` against
`adw-factory` resolves real PR state for every banked run, and `gh auth logout`
leaves the board byte-identical minus the chips.

**Stage 3 — the chips.** `CardSummary` gains its optional PR fields;
`renderTicketCard` renders both axes with all four absent-states. *Done when:*
the board shows CI and validation for every card that has a PR, and a TTL
refresh pushes the change over SSE without a reload.

## Further Notes

**This is half of "validation", and the other half is landing right now.**
At the time of writing, `src/pipeline/lanes/shared.ts` has **uncommitted** work
importing `makeReviewGateLoop` and splicing the two review nodes into the
shared tail between `gates` and `commit`, behind a `deps.reviewEnabled` flag:

```
makeGatesNode() → [review:standards, review:spec] → makeCommitNode()
```

So `adw-m9-06`'s wiring is in flight, not deferred. As soon as it commits, runs
begin journaling `node-end` events with `details.kind === "review"` (a shape
`journal.ts:178` has typed since m9-01) and the card gains access to the
*factory's own* two-axis verdict — from the journal it already reads, with zero
GitHub calls and zero network dependency. Today that count is still **0 of 18**
banked journals.

**Consequence for this spec: the card will soon have two review signals, and
they must not be conflated.** The agent's pre-PR verdict answers *"did the code
meet our standards and the ticket's spec before we asked anyone to look"*; the
human's post-PR verdict answers *"did a person accept it"*. They can disagree,
and a card that merges them into one "reviewed" chip destroys exactly the
information that disagreement carries.

v1.9 still ships the GitHub axis first — it is unambiguous today and it is the
one that is silent on a team repo. But Stage 3's chip layout **should reserve
the slot** for the factory verdict rather than claim the row, so the follow-up
is an addition and not a redesign.

> **Verify before building.** That wiring was uncommitted when this was
> written. Re-check `grep -n makeReviewGateLoop src/pipeline/lanes/shared.ts`
> and `grep -c '"kind":"review"' runs/*/journal.jsonl` at pickup — if it has
> landed and journals now carry verdicts, revisit whether Stage 3 should ship
> both axes at once.

**v1.2 has drifted once already.** Its §Out-of-Scope rules out cost display
*"under the operator's subscription economics, marginal dollar cost is ≈ 0"* —
and `renderTicketCard` ships a `$` KPI. Not this spec's job to resolve, but it
should be either amended or reverted rather than left standing as a spec that
contradicts the code it governs.

### Operator amendment 2026-09-29 — only active targets, and the filter applies

§Implementation-Decisions 2's "one `gh pr list` per target" is narrowed to
**one per *active* target**: a target is active when its ticket store holds at
least one `.md` ticket, re-read once per refresh cycle. An inactive target is
never fetched and raises no notice — 10 of the 15 configured targets were
filling the board with "no GitHub repo resolvable" lines for projects with no
tickets and no runs. Per-target PR notices also ride a `noticeTargets` map
(notice text → target) so the board's project filter hides the others.
`adw web` now carries the same stale-`GITHUB_TOKEN`/`GH_TOKEN` refusal as
`run`/`sync`: a stale exported token turned every target's fetch into a 401.
