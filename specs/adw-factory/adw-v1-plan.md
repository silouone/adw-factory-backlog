# Plan: adw-factory v1 — the chore lane

> Phase 2 — Plan. HOW. Every decision traces back to a requirement in `adw-v1.md`.

- **Spec:** adw-v1
- **Status:** draft — awaiting operator approval before Tasks phase

## 1. Specification summary

One lane (chore) end-to-end for target repo cLens: markdown ticket in the target
repo → deterministic validate/dispatch → isolated workspace → build agent →
lint/typecheck/test fail-loop (3 local rounds, same session) → push + real PR →
1 CI repair round → human merges. Three observability layers: the journal
from run #1; cLens capture and vendor-neutral traces land at M3, verified by
a replay audit — runs before M3 exit are construction-phase validation runs,
outside success-metric 3 *(clarified 2026-07-14, post-review)*. Serial manual CLI, worktree
isolation first with container and remote behind the same interface,
subscription auth, mid-tier workhorse model, sky-high time/turn caps so rounds
govern.

## 2. Constitution gates (Phase -1)

| Gate | Pass? | Justification if not |
|------|-------|----------------------|
| I Test-first | ☑ | Factory built TDD with `bun test`; blueprint engine and intake are pure-function-heavy and highly testable. |
| II Simplicity | ☑ | One lane, one target, three modules (intake, pipeline, observability) + CLI shell. Container/E2B are later milestones, but the `Workspace` interface exists day one — justified: retrofitting isolation seams is the one rework the grilling explicitly priced. |
| III Code > agents | ☑ | Only 2 agent node types (build, repair-resume); all else deterministic. |
| IV Humans at ends | ☑ | No merge/approve capability exists in the codebase at all. |
| V Bounded autonomy | ☑ | 3 local + 1 CI rounds; time/turns ceilings present, defaults high. |
| VI Observable | ☑ | Journal writer + tracer are pipeline middleware — a node cannot run un-observed by construction. |
| VII Isolation | ☑ | Agent `cwd` is always a provisioned workspace; push happens only from the gate-green workspace. |
| VIII Anti-abstraction | ☑ | Agent SDK, `git`, `gh` called directly. The single deliberate interface is `Workspace` (three real implementations planned = not speculative). |
| IX Craft | ☑ | Strict TS, pure core, I/O at edges, errors carry `ticketId` + `node`. |

**Complexity tracking:** nothing failed. Watch item: the OTel layer is the most
plumbing-heavy piece; it stays a middleware that can be disabled by a no-op
exporter, never a dependency of pipeline logic.

## 3. Architecture

```
~/adw-factory (Bun + strict TypeScript)
├── src/
│   ├── cli.ts                  adw run [--ticket <id>] [--isolation worktree|container|remote]
│   │                           adw clean [--ticket <id>] · adw status
│   ├── intake/                 ticket parse (frontmatter), schema validation,
│   │                           dispatch (switch on type → lane), status writes (commit each)
│   ├── pipeline/
│   │   ├── engine.ts           blueprint state machine: Node[] runner, per-node
│   │   │                       journal+span middleware, abort controller (SIGINT, ceilings)
│   │   ├── lanes/chore.ts      node list + lane config (models, rounds, caps)
│   │   └── nodes/              deterministic: provision, assemble-prompt, gates,
│   │   │                       push, open-pr, sync-pr-state, status
│   │   │                       agent: build (SDK query()), repair (query({resume}))
│   ├── workspace/
│   │   ├── types.ts            Workspace: provision → exec → collect → teardown
│   │   ├── worktree.ts         M1 — git worktree + branch + bun install
│   │   ├── container.ts        M4 — OrbStack/Docker, prebaked bun+git+claude image
│   │   └── e2b.ts              M5 — E2B custom template, key from env
│   ├── observability/
│   │   ├── journal.ts          JSONL append, one file per run
│   │   ├── tracing.ts          OTel-native engine (pattern lifted from
│   │   │                       translated-language-service src/common/lib/tracing.ts)
│   │   ├── span-exporter.ts    OTLP-JSON → injected sink (filesystem in v1)
│   │   └── clens.ts            capture-hook injection + ADW_TICKET_ID tagging
│   └── targets/                loader for per-target config
├── targets/clens.json          the thin target config (see §5)
├── prompts/chore-build.md      versioned prompt templates (+ repair-report template)
├── specs/                      this folder
└── test/                       bun test, mirrors src/
```

**Blueprint engine.** A lane is an ordered list of nodes; each node is
`(ctx) => Promise<NodeResult>` where `NodeResult` routes `next | retry(payload) |
fail(reason)`. The engine owns retry accounting (rounds), ceilings, journal
entries, and span open/close — nodes stay pure logic. Agent nodes call
`query()` from `@anthropic-ai/claude-agent-sdk` with: `cwd` = workspace path,
per-node `model`, `permissionMode`/allowed-tools scoped to the workspace,
`env` carrying `TRACEPARENT` + `ADW_TICKET_ID`, hooks = cLens capture. Repair
nodes pass `resume: <sessionId>` captured from the build node's init message.

**Chore lane node graph.**

```
validate → dispatch → provision(worktree) → assemble-prompt
  → build(agent) → gates(code: lint→typecheck→test)
       ↑ fail (≤3, same session) ↓ pass
  → push → open-pr → status(in-review)
  → [next run] sync-pr-state: ci-fail→repair(agent,≤1)→push ;
                merged→done ; closed→rejected(reopen)
```

**Auth & billing.** SDK rides the machine's Claude login (or
`CLAUDE_CODE_OAUTH_TOKEN` from `claude setup-token` for container/remote).
`ANTHROPIC_API_KEY` intentionally unset; moving to API later is config.

## 4. Decisions & rationale (the 14 locked decisions)

| # | Decision | Serves requirement | Rationale / alternatives rejected |
|---|----------|--------------------|------------------------------------|
| 1 | Standalone repo `~/adw-factory` | NFR target-agnostic | Factory outlives customer #1; embedding in cLens ships private tooling in an OSS package. |
| 2 | Chore lane only | Story 1–7 scope | Prove all shared plumbing on the cheapest lane; lanes multiply on proven rails. |
| 3 | Markdown tickets, one file each, in target repo `tickets/` | Story 1, NFR audit log | Frontmatter = machine contract; git = audit; per-file = no write conflicts. GitHub Issues demoted to future importer. |
| 4 | TS Agent SDK on Bun, blueprint state machine | Stories 1–2, NFR $0 | Stripe-validated pattern; typed events, session resume, hooks in code. Headless `claude -p` = same billing, worse contract. |
| 5 | `Workspace` interface; worktree → OrbStack container → E2B | Story 4 | Isolation is an operator param; never debug pipeline and infra simultaneously. E2B beat Daytona/Modal/Blaxel on TS SDK maturity + key-in-hand. |
| 6 | Code-only dispatch on `type:` | Story 1 | Routing is degenerate with one lane; router agent arrives with lane #2. |
| 7 | 3 local repair rounds resuming same session, then 1 CI round | Story 2 | Talk: same session keeps working context. Stripe calibration: hard ceilings, CI as confirmation. |
| 8 | Real PR, human merges, status follows PR lifecycle | Story 3 | "An agent cannot be held accountable." Track record before ceremony is automated. |
| 9 | Journal + cLens capture + OTel spans | Story 5 | Three layers, distinct consumers: factory debugging, product dogfood, backend-agnostic future. |
| 10 | Sonnet workhorse, model per node, frontmatter override | Story 6, NFR cost | Rate-limit thrift where judgment is low; SOTA reserved for future planner nodes. |
| 11 | Manual serial CLI | Story 6 | Operator is the scheduler while trust builds; watcher/parallel are flags later. |
| 12 | Time+turns caps implemented, defaults sky-high | Story 2 | Rounds are the intended governor; tighten from journal data, not guesses. |
| 13 | Light context pack (ticket + config + pointers) | Story 1 | Native discovery suffices for chores; grep-pack only if journals prove discovery cost. |
| 14 | Name `adw-factory` (working title) | — | Swap when a better name is probed; rename is cheap for a personal tool. |

## 5. Data model & contracts

**Ticket** (`tickets/<id>.md` in target repo):

```yaml
---
id: clens-042            # unique within target
type: chore              # v1: only 'chore' has a registered lane
status: queued           # queued|in-progress|in-review|done|blocked|rejected
priority: 2              # 1 high … 3 low; tiebreak = oldest first
created: 2026-07-14
model: sonnet            # optional per-ticket override
caps: { minutes: 480, turns: 200 }   # optional; defaults are deliberately huge
attempts: []             # appended by factory: branch, pr, outcome per attempt
---
Free-form body: what to do, acceptance hints, constraints.
```

Status transitions are written by code and committed one-by-one
(`adw: clens-042 → in-review`). `blocked`/`rejected` re-enter via operator edit
back to `queued`; reviewer comments are appended to the body on rejection.

**Status-commit mechanics (added 2026-07-14, post-review):** status commits
land on the target's **default branch in the primary working copy**, staging
only `tickets/<id>.md` (other dirty files untouched, per spec §5). Dispatch
requires the default branch checked out there; otherwise the factory refuses
with a descriptive error. The dispatch check+commit pair is serialized by an
exclusive lockfile (`runs/locks/<ticketId>.lock`, `O_EXCL`) so concurrent
invocations can never dispatch the same ticket. Attempt branches are cut from
`origin/<base>` when a remote exists (local `<base>` in fixtures), so status
commits never appear in PR diffs. The factory never pushes the default
branch; status history reaches the remote via the operator's own pushes.
Every terminal `blocked`/aborted path ends with a committed `status: blocked`
recording run id, attempt branch, and workspace — no path leaves a ticket
`in-progress`.

**Target config** (`targets/clens.json` in factory repo):

```json
{
  "name": "clens",
  "repo": "/Users/silouane/agent-observability-project",
  "base": "main",
  "branchPrefix": "adw/",
  "gates": [
    { "name": "lint",      "cmd": "bun run lint" },
    { "name": "typecheck", "cmd": "bun run typecheck" },
    { "name": "test",      "cmd": "bun run test" }
  ],
  "setup": "bun install",
  "context": ["CONTEXT.md", ".claude/skills/coding-standards/SKILL.md"],
  "ci": { "provider": "github-actions", "poll": "gh pr checks" }
}
```

**Journal** (`runs/<runId>/journal.jsonl` in factory repo, gitignored):
one event per line — `run-start`, `node-start`, `node-end` (with gate results /
agent usage: tokens, turns, sessionId), `round`, `abort`, `run-end` (outcome).
PR body stats are computed from this file.

> **[AMENDED] 2026-09-28**, ticket
> `adw-usage-07-price-codex-runs-and-journal-the-rate-713388`: a `node-end`'s
> agent usage may now carry a `rate` — the four USD/MTok fields
> (input/output/cacheRead/cacheWrite) plus `asOf` and, where a promotional
> rate applies, `promoUntil` — the price the factory resolved for that
> node's model AT THE TIME IT RAN (`src/rate-card.ts`'s
> `resolveJournaledRate`). Chosen over re-pricing from a live rate card at
> render time: a promotional rate (e.g. gpt-5.6-sol, promo through
> 2026-11-21) can expire, and re-pricing an old run from today's card would
> silently reprice history it wasn't true of. `estimateUsd` (src/web/
> pricing.ts) prices from this journaled rate first, falling back to the
> live card only for journals banked before this field existed. Absent when
> the node's model was unpriced (rule 2: unknown means no estimate).

**Traces** (`runs/<runId>/spans/`): OTLP-JSON `ExportTraceServiceRequest` files
via an injected `SpanExporter` (filesystem sink v1). One trace per run; root
span = ticket run; child span per node; agent spans typed `AGENT`, gate/git
spans `TOOL`. `TRACEPARENT` env propagates into agent processes so container/
remote sessions parent correctly (W3C context, exactly the TLS cross-process
pattern); the engine exposes the active node's context as `ctx.traceparent`
and the agent nodes merge it over `agentEnv()` — the workspace contract stays
tracing-unaware.

**PR body template** (`prompts/pr-body.md`): ticket id + link, summary (agent-
written, one paragraph), gate table, rounds used, duration, run id, cLens
session ids.

**Workspace contract** (`src/workspace/types.ts`):

```ts
interface Workspace {
  readonly kind: 'worktree' | 'container' | 'remote'
  readonly path: string                    // agent cwd
  exec(cmd: string, opts?): Promise<ExecResult>   // gates, git, setup
  agentEnv(): Record<string, string>       // ADW_TICKET_ID, auth (TRACEPARENT is merged per-node by agent nodes, not here)
  teardown(): Promise<void>                // clean only; abort keeps it
  readonly agentSpawn?: AgentSpawnSpec     // M4 amendment: how agent processes
                                           // are spawned for this kind; absent
                                           // → SDK default local spawn (worktree)
  push(branch: string): Promise<ExecResult> // M4 amendment #3 (adw-m4-07,
                                           // operator-approved 2026-07-18):
                                           // publish the gate-green attempt
                                           // branch to origin — the ONE
                                           // credentialed push I/O, kind-owned.
                                           // The push node keeps its guards
                                           // (S2.7, dirty/base/detached, E6
                                           // retry bound) and delegates the
                                           // push itself. Worktree: in-place
                                           // `git push -u origin <branch>`
                                           // (host keyring), byte-identical to
                                           // before. Container: export the
                                           // gate-green commit out of the
                                           // container and push from a fresh
                                           // factory-owned HOST-side context
                                           // (hooks disabled, no workspace-
                                           // local config, provision-captured
                                           // validated origin URL, token host-
                                           // side only) — the push credential
                                           // and the origin never touch the
                                           // agent-writable checkout, and the
                                           // rw local-origin mount is gone
                                           // (S4.2). REPLACES pushEnv?()
                                           // (amendment #2, adw-m4-03): after
                                           // the move no kind layers a push
                                           // credential onto in-workspace
                                           // execs, so the channel is removed
                                           // rather than left as a dead
                                           // credential path.
  hardStop?(): Promise<void>               // M4 amendment #4 (adw-m4-06,
                                           // operator-approved 2026-07-18):
                                           // real OS-level hard stop for a
                                           // deadline/abort breach (Art. V).
                                           // Container: `docker kill` (SIGKILL
                                           // PID-1) after the SDK's graceful
                                           // stdin-EOF grace — stops a wedged/
                                           // runaway agent the abortController
                                           // can't (client-kill doesn't cross
                                           // into the container). Leaves the
                                           // container EXITED, not removed, so
                                           // Art. VII (kept for autopsy) holds
                                           // and `adw clean` stays the only
                                           // reclaimer. Absent → graceful
                                           // abort only (worktree: the agent
                                           // is a host child the SDK abort
                                           // already reaches).
}
// M4 amendment #5 (adw-m4-06): ExecOptions gains an optional `timeoutMs` — a
// command exceeding it is killed and returns a non-zero ExecResult naming the
// timeout, so a hung in-container gate/git command cannot wedge the lane past
// the wall-clock deadline (the awaited exec would otherwise never return to a
// boundary check). Generous, configurable default shared by both kinds; a
// legitimately long gate (a full test suite) must not trip it.
interface ExecOptions {
  readonly env?: Record<string, string>
  readonly timeoutMs?: number              // M4 amendment #5 (adw-m4-06)
}
// M4 amendment (operator-approved 2026-07-16, adw-m4-02 spike): a pure
// descriptor the agent nodes forward through AgentQueryOptions ONLY when
// present (worktree options stay byte-identical — the M3 hooks/traceparent
// precedent); live-query.ts maps it onto the SDK's spawnClaudeCodeProcess
// (`docker exec -i` — claude runs INSIDE the container, S4.5).
//
// M4 amendment #6 (operator-approved 2026-07-18, adw-m5-02; the m5-01 spike
// PASSED the gate): AgentSpawnSpec becomes a DISCRIMINATED UNION so the remote
// (E2B) kind rides the same seam. The e2b variant carries a `sandboxId`, and
// live-query maps it onto spawnClaudeCodeProcess via an E2B `SpawnedProcess`
// adapter over `commands.run({background: true, stdin: true})` — stdin →
// `sendStdin`, graceful stop → `closeStdin` (real stdin-EOF, spike-proven),
// kill → `commands.kill`; the spawned command is the IMMUTABLE
// `/opt/claude/share/versions/<ver>/claude` absolute path (executable-integrity,
// the Codex Critical fix). The spec stays PURE DATA (no live Sandbox handle):
// the adapter reconnects from `sandboxId` lazily, exactly as the container
// mapping builds a stateless `docker exec` argv from `containerId` — so both
// kinds stay unit-testable without a network. Env crosses via the command's
// `envs`, never argv (the M4 ps-visibility rule).
type AgentSpawnSpec = ContainerSpawnSpec | E2bSpawnSpec
interface ContainerSpawnSpec {
  readonly kind: 'container'
  readonly containerId: string
  readonly workdir: string                 // in-container agent cwd (== path)
}
interface E2bSpawnSpec {
  readonly kind: 'e2b'
  readonly sandboxId: string               // reconnected from, not a live handle
  readonly workdir: string                 // in-sandbox agent cwd (== path)
}
```

**Amendment #7 (operator-approved 2026-07-19, adw-m7-01; provider-shaped
`AgentQuery` boundary).** `AgentQuery` (`src/pipeline/nodes/build.ts`) is a
structural mirror of the Claude Agent SDK's `query`, with `src/live-query.ts`
its single binding. Multi-provider makes **provider** a dimension **orthogonal
to isolation kind** (M4/M5): the same injected seam gets a second binding,
`src/codex-query.ts`, mapping `codex exec --json` JSONL (`thread.started` → init
`session_id`; `turn.*`/`item.*` → assistant turns + result with usage) onto the
**unchanged** `SdkMessage` mirror. **Provider selection is which binding the CLI
injects** — chosen by **target config** (`provider?: 'claude' | 'codex'`,
default `'claude'`) or a CLI `--provider` flag; **NOT per-ticket frontmatter**
(the token-outage case is a whole-run switch). Repair **resume** maps
`codex exec resume <session_id>` onto `AgentQueryOptions.resume`. The Claude-only
`CHORE_MODEL: 'sonnet'` lane default becomes a **per-provider default model**
(each provider names its own; ticket `model:` override still wins, S6.3).
`AgentQueryOptions` and `SdkMessage` are **unchanged** — the mirror was already
SDK-neutral; the only additions are the CLI's provider selection +
`codex-query.ts`. **Worktree kind only**: container/e2b Codex parity (owning the
docker-exec / e2b stdio JSONL transport, re-deriving the m4/m5 teardown
hardening, a `~/.codex/auth.json` refresh story) is **Tier-2, explicitly
deferred**. `env-policy.ts`'s `SENSITIVE_ENV_KEY` already matches
`OPENAI_API_KEY` (`_KEY` suffix); the amendment confirms Codex/OpenAI credential
coverage and adds any exact-name vars that escape the suffix families. Accepted
degradations: coarser Codex permissioning (`--sandbox workspace-write` vs
Claude's `Bash` allowlist), caps recalibration (`turns` is Claude-message-scale),
hooks-based capture (adw-m7-02) instead of SDK hooks.

## 6. Risks & mitigations

- **Rate-limit exhaustion mid-run** → graceful abort to `blocked` (spec §5);
  serial operation + Sonnet default keeps burn low; journal names the cause.
- **Repair loop thrash on misunderstood failures** → hard 3-round ceiling; the
  structured failure report includes the *diff so far* to anchor the agent.
- **PR-state sync drift** (merged/closed while factory idle) → `sync-pr-state`
  runs at the start of every invocation, before ticket selection.
- **cLens hooks in workspaces** — worktrees only contain committed files; if the
  target's capture settings aren't committed, injection happens via SDK hook
  options instead. Verify at M3 with a live capture check.
- **Container auth** (M4) → `claude setup-token` output mounted as env; never
  baked into the image.
- **Prompt quality unknowns** → prompts are versioned files; every run's journal
  + captured session makes prompt regressions diffable and measurable.
- **Ticket race / double dispatch** → status commit is the lock: dispatch
  requires `queued` at HEAD and commits `in-progress` before provisioning.
- **Lease refused on the attempt branch** (operator pushed to it while the PR
  was open) → the rebase round ends `open` with a `lease-refused` reason, the
  local rebase is left in place for autopsy, and the push is never retried or
  forced.

## 7. Coverage check

Every acceptance criterion in `adw-v1.md` maps to a component above: Story 1 →
intake + chore lane nodes; Story 2 → engine retry accounting + gates node +
repair node; Story 3 → sync-pr-state + absent merge capability; Story 4 →
Workspace interface + M1/M4/M5; Story 5 → observability trio; Story 6 → CLI +
lane config; Story 7 → shakedown milestone; NFRs → auth choice, target config,
committed status transitions; edge cases → §6 mitigations + engine abort paths. ☑

> **PROPOSED amendment (ticket `adw-resume-01-continue-a-blocked-run-9ae0cb`;
> not in force until the operator approves it)** — companion to the Story 6
> criterion in `adw-v1.md`. A resumed run is a new runId carrying
> `run-start.resumedFrom` and `Attempt.resumedFrom`; journals, the board and §6
> metrics stay one-run-one-outcome. The workspace stays
> `runs/<oldRunId>/workspace`. The restart point is the last lane `node-start`
> in the old journal (`repair` → gates, review-fix → its review node). Round and
> turn counters restart at 0. Worktree isolation only. The bug lane refuses to
> resume past `red-check`. No base re-check on resume (the tail's rebase node
> covers a moved base). No `--from` in v1; `resumeLane(lane, seed)` takes
> `startAt` explicitly as the seam.

> **PROPOSED amendment (ticket `adw-bug-33-repair-rounds-are-charged-for-a-run-that-changed-nothing-23bd0f`;
> not in force until the operator approves it)** — a third terminal run
> outcome, `deferred`, beside `green` and `blocked`. A build or plan agent that
> refuses to start (prerequisite not done) says so in a structured header at
> the top of its own artifact file (`.adw/artifacts/<node>.md`), never in
> prose the factory would have to match:
> `---` / `status: not-started` / `reason: <one line>` / `---`. The node
> returns a `fail` carrying `deferred: true`; the engine journals `run-end`
> with `outcome: "deferred"` and the agent's reason, runs nothing after it
> (no `test`, `gates` or repair round), and the CLI returns the ticket to
> `queued` (`adw: <id> → queued`, no attempt recorded — nothing was tried).
> A `deferred` run exits `EXIT_BLOCKED` (non-zero: no PR was opened). Separately,
> a gate failure on a tree identical to base (empty `git diff <base>`,
> tracked and untracked, `.adw/` excluded) is base-red by construction: the
> gates node ends the run `blocked` with `baseRed: {gate, sha}` journaled
> as a `base-red` event instead of charging a repair round.

## 8. Milestones (Tasks phase will decompose)

- **M1 — Loop proven:** CLI, intake, worktree workspace, assemble-prompt, build
  agent, gates, 3-round repair loop, journal. Exit: a toy chore reaches a green
  local branch.
- **M2 — Lifecycle complete:** push, PR open, CI round, sync-pr-state, status
  transitions end-to-end. Exit: toy chore merged by operator, ticket → done.
- **M3 — Observability trio:** OTel engine + filesystem exporter + traceparent
  into agents; cLens capture tagging verified live. Exit: run replayable from
  artifacts alone.
- **M6 — Shakedown** *(reordered before M4/M5, 2026-07-14 post-review)*:
  ≥3 real cLens chores merged on worktree isolation; success metrics measured
  from journals; tighten caps/defaults from data. Exit: v1 done. Rationale:
  real-customer proof before isolation expansion (Art. II), and spec S4.5
  already reads this way — v1 ships worktree; container/remote are
  "delivered next".
- **M4 — Container workspace** (OrbStack image, auth mount) — post-v1
  expansion.
- **M5 — Remote workspace** (E2B template) — post-v1 expansion.
