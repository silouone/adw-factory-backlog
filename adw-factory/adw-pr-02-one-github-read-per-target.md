---
id: adw-pr-02-one-github-read-per-target
type: feat
status: in-progress
priority: 2
created: 2026-09-17
caps: {minutes: 150, turns: 800}
depends: [adw-pr-01-resolve-a-card-to-its-pull-request]
attempts: [{"runId":"adw-pr-02-one-github-read-per-target-1789738866494","branch":"adw/adw-pr-02-one-github-read-per-target","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-pr-02-one-github-read-per-target-1789738866494/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# One GitHub read per target, behind a TTL — and a web server that knows its targets

Spec: `specs/adw-v1.9-pr-state.md` (Stage 2). Measurements:
`ai_docs/2026-09-17-github-ci-review-on-card.md` §6.

Stage 2 of three, and **the risk stage of the whole feature**. Stage 1 built
the pure derivations; this ticket gives them data. Still nothing visible on the
board — Stage 3 renders. What lands here is a PR index the board can ask, and
the first time `adw web` has ever known what a target is.

## The cost constraint decides the architecture

Measured on 2026-09-17: rate limit **5 000/hour**; `gh pr view` ≈ 0.85 s;
`gh pr list --state all --limit 100` with the full field set ≈ 3.16 s for
38.7 KB covering all 60 PRs. `SSE_POLL_MS = 1_000` (`server.ts:135`), and every
tick re-lists and re-projects every card.

| Strategy | Calls/hour | Verdict |
|---|---|---|
| `gh pr view` per card on the SSE tick (10 cards) | **36 000** | 7.2× over — dead in 8 min |
| `gh pr list` per target on the SSE tick (11 targets) | 39 600 | worse |
| **`gh pr list` per target, 60 s TTL** | **660** | **13 % of budget** |

**The GitHub refresh must be decoupled from the journal poll.** The SSE tick
*reads* the cache and never triggers a fetch.

A slow refresh is therefore **not** a page stall: the 17 s worst case measured
below sits on the background TTL timer, so it delays freshness by seconds and
blocks no request.

- [ ] `gh` is **never** called on the request path or the SSE tick. Assert it.

## 1. The PR index edge

- [ ] One query per target:
      `gh pr list --repo <owner/name> --state all --limit 300
       --search "head:<branchPrefix>" --json
       number,state,headRefName,statusCheckRollup,reviewDecision`
      — addressed with `--repo` from Stage 1's resolver, **never relying on
      cwd**, since the 7 Coorp targets are not cloned.
- [ ] **The `--search "head:<branchPrefix>"` scoping is not optional.**
      Measured 2026-09-17 against `CoorpAcademy/coorpacademy` (**13 359 PRs**):
      an unscoped `--limit 300` with `statusCheckRollup` took **17.1 s /
      180 KB**; the same query scoped to `head:adw/` took **0.7 s**.
      `statusCheckRollup` is a nested per-PR fetch, so cost scales with rows
      returned. Unscoped, coverage is also *wrong* — 100 newest of 13 359 is
      roughly a week of traffic on that repo, and a factory PR left unreviewed
      for a fortnight silently drops out of the window.
- [ ] **A search-hostile `branchPrefix` must fail loudly**, never silently
      widen to every branch in the repo. All 11 targets use `adw/` today;
      validate the prefix is safe to interpolate into a `head:` qualifier and
      refuse with a descriptive error (Art. IX) if it is not.
- [ ] **Bound the window and say when you hit it.** `--limit 300` with rows
      == limit means older factory PRs were dropped → notice. Measured
      2026-09-17: this repo has **68** factory PRs growing at **~10/day**
      (peak 24), so it crosses 100 within days and 300 within weeks. The bound
      is self-limiting — it grows only as fast as the factory runs — but it is
      not infinite.
- [ ] **Named escape hatch, for when 300 stops being enough:** the board only
      ever renders the tickets in `runs/` (18 today), so the query can be
      narrowed to exactly those ids rather than a time window. Do **not**
      build it now; record it here so the next person does not re-derive it.
- [ ] The runner is **injected**, mirroring `GhRunner` (`(args) => stdout`,
      throws on failure). Bound by the CLI over `liveGh`; a scripted fake in
      tests. **No test touches the network.**
- [ ] Reuse `cli.ts::interpretGh`'s exit tolerance rather than re-deriving it.
- [ ] Rows are joined to ticket ids with Stage 1's join and keyed by ticket.
- [ ] `--limit 100` is a **bound, and bounds get named** — if a repo returns
      100 rows, say so on a notice rather than silently showing stale chips for
      older tickets.

## 2. The TTL cache

- [ ] TTL **60 s**, a named constant carrying its rationale in source, the way
      `HEARTBEAT_STALE_MS` already does.
- [ ] Clock **injected** (Art. IX) — no argless `Date.now()` inside the module.
- [ ] A read inside the TTL issues **no** call. A read past it issues exactly
      **one** per target, never one per card.
- [ ] A cache miss returns "not fetched yet" immediately — it never blocks the
      response. Same tolerance stance `loadRuns` already takes.
- [ ] Refresh must not stampede: concurrent readers past the TTL trigger one
      fetch, not N.

## 3. Degradation is a requirement, not an afterthought

The board is network-free today. This ticket makes it *optionally* networked,
and that is the real amendment in v1.9 — so the failure path is part of the
deliverable, not a nicety.

- [ ] `gh` missing, logged out, rate-limited, offline, or throwing for any
      reason → an **empty index plus a notice naming the target and the
      reason**. Never a partial index, which would read as "this repo has no
      PRs".
- [ ] No repo identity resolvable (no `github` field, no clone) → that
      target's cards simply carry no PR data, with a notice. Not an error.
- [ ] **`gh auth logout` must leave the board fully usable** — every run, every
      Gantt, every KPI unchanged. This is the acceptance bar for the whole
      ticket.

## 4. `runWeb` gains target awareness — expect this to bite

`runWeb` (`src/cli.ts:1215`) takes `{ runsRoot }` and is deliberately
target-agnostic. Every *other* `adw` command resolves **one** target via
`--target`; the board renders **all** of them, grouped by `run-start.target`.
So this needs `targetsDir` plus a load-**every**-config path that exists
nowhere in `cli.ts` today.

- [ ] `WebDeps` gains `targetsDir`; `runWeb` loads every target config once at
      startup and indexes them by name.
- [ ] **A malformed target config must not kill the server.** It is named on a
      notice and its cards go without chips — the same tolerant-read stance
      `loadRuns` takes for an unreadable journal.
- [ ] **Decide and document: a `runs/` dir whose recorded `target` has no
      config file at all.** The spec flags this as the most likely gap. Also
      handle `target ?? "unrecorded"` (`render.ts`) — measured 0 of 18 banked
      runs today, but the code path is live.
- [ ] Config is loaded at startup, not per request.

## 5. Read-only must provably survive

- [ ] `sync-pr-state.ts`'s negative-capability guard — which proves no mutating
      PR command is constructed anywhere in `src` — is **extended to cover the
      new module**, not left behind.
- [ ] `test/web/server.test.ts:204`'s grep test (no `"POST"`/`"PUT"`/
      `"DELETE"`/`"PATCH"` literal in `server.ts`) stays green untouched.
- [ ] A `gh pr list` is a read. Nothing here approves, merges, closes,
      comments or re-runs anything.

## TDD (Art. I — non-negotiable)

Tests first, red, reviewed, then green. The cache tests use a **fake clock and
a fake runner** and assert call *counts*, which is the only way the cost
constraint above is actually enforced rather than merely intended:

- a second read inside the TTL issues **0** calls
- a read past the TTL issues exactly **1** per target
- N targets → N calls, **never N × cards**
- a throwing runner → empty index + notice, never a partial one

Bank a real `gh pr list` payload from this repo as the fixture.

## Verify

- [ ] Every new suite green; call-count assertions present and meaningful.
- [ ] Live: `adw web` against `adw-factory` resolves real PR state for every
      banked run that has a PR.
- [ ] Live: `gh auth logout`, reload — the board is byte-identical minus the
      PR data, with a notice. Log back in, wait out the TTL, data returns.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

All rendering — Stage 3 (`adw-pr-03`). Per-PR `gh pr view` for review
*feedback text* (deferred in the spec). Journaling the PR URL on `open-pr`
(worth its own ticket; the branch join makes it unnecessary here).
