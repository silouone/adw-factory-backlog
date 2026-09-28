---
id: adw-gh
type: epic
status: queued
priority: 1
created: 2026-09-28
depends: []
children:
  - adw-gh-01-the-suite-never-reaches-real-github
attempts: []
---
# EPIC — GitHub is a shared, metered resource: publishing a PR is an event, not a step that can fail the run

**Needs a spec before children beyond `-01` are cut.** The outbox changes
the lane's terminal shape (open-pr stops being a node that can block) and
adds a ticket or attempt state, so it amends `adw-v1.md` §2 and
`adw-v1-plan.md` §5. Operator decision.

## What happened (2026-09-28, measured)

Two runs finished all their work: gates green, review done, branch pushed.
Both were then **blocked at `open-pr`**:

- `adw-graph-02`: 62 attempts in 157 s
- `cqc-fe-12`: 66 attempts in 154 s

Both got the same answer every time: `GraphQL: API rate limit already
exceeded for user ID 29061867`. That was 90–105 minutes of agent work, about
$29, thrown into `blocked` by a quota that reset about an hour later. The
operator salvaged both by hand (PRs #153, cmc#144).

**Who burned the quota:** the in-flight `adw-pr-02` run's own test suite.
Evidence strength:

- **Observed at 20:46Z:** `gh` processes whose parent was that worktree's
  `bun test`.
- **Inferred for the two failure windows (20:11–20:13Z, 20:33–20:36Z):**
  the run was in `build` (19:51–20:49Z), where the agent runs `bun test`.
- `main` has no PR-index poller, so the `adw web` boards are ruled out.
Its new PR-index wiring defaults to a real `gh` spawn
(`defaultPrIndexGh`, `src/web/server.ts:469` in that run's worktree), and
the existing `startWebServer(...)` tests don't inject `prIndexGh`. So every
`bun test` a gate or the agent ran fired
`gh pr list --limit 300 … statusCheckRollup` at 5 live repos. The process
tree showed PID `bun test --reporter=junit` as the parent of the `gh`
calls. GraphQL went to **0/5000** while the REST `core` quota stayed at
**5000/5000**.

Two lessons:

1. Nothing stops a test from reaching the real network. See `adw-gh-01`.
2. The factory has one budget holder (one user token) and N uncoordinated
   spenders: every lane's `open-pr`, every `sync-pr-state`, every CI-round
   check read, the board's PR index, and an agent's own `gh`. None of them
   knows what the others are spending.

## Why backoff is the wrong shape

`retry-policy.ts` (adw-bug-09) was built for a **blip**: 90 s of summed
delay (`DEFAULT_RETRY_BUDGET_MS`), then fail. A rate limit is not a blip.
It is a known quota with a **known reset time** (`rateLimit.resetAt`,
`x-ratelimit-reset`). Retrying inside that window can't succeed and only
spends more of the quota. At scale (many lanes, PRs landing everywhere) a
throttle will be routine, not exceptional, so the design must make it
harmless, not rare.

## Direction (to be specced)

1. **Publishing is an outbox.** On a green lane, `open-pr` stops calling
   GitHub. It writes a durable **publish intent**, idempotent by
   `(repo, head)`:
   - repo, base, head, title, body, ticketId, runId
   - written to e.g. `runs/outbox/<ticketId>.json` plus a journal event
   The run then ends **green: published-pending**. A GitHub outage or
   throttle can no longer turn finished work into `blocked`.
2. **A single publisher drains it, event-driven.** It wakes on:
   - a new intent
   - quota reset (a timer armed at `resetAt`)
   - operator nudge (`adw publish`)

   Before each create it reconciles by head branch (existing
   `gh pr list --head` logic, lost-response safe). Then it transitions the
   ticket to `in-review` with the PR url, the same status-commit protocol as
   today.
3. **One GitHub broker per host holds the budget.** Every factory call goes
   through it; it reads the remaining quota from response headers and knows
   `resetAt`. It has priorities:
   - writes (publish, push-adjacent) first
   - then CI/sync reads
   - then board reads, which are the first to be shed
4. **Pick the API by quota.** `gh pr create` is GraphQL. The REST
   `POST /repos/{o}/{r}/pulls` spends the separate `core` quota, which is how
   today's salvage worked while GraphQL was at 0. The broker picks whichever
   quota has headroom.
5. **Agents don't spend the factory's quota.** An agent `gh` in a lane
   should go through the broker or be denied (allowlist is bun/bunx today,
   so check the Codex path too).

## Done when (epic)

- Driving the GraphQL quota to 0 mid-run ends the run green-pending, not
  blocked. The PR opens within one minute of `resetAt`, with no operator
  action.
- Journals prove a burst of N simultaneous green lanes opens N PRs, none
  duplicated.
- The board's GitHub reads can be starved without delaying any publish.
