---
id: adw-m7-01-codex-worktree
type: feat
status: done
priority: 3
created: 2026-07-19
epic: adw-m7
depends: [adw-m1-11-integration]
attempts: []
---
# Codex provider on the worktree lane

> Refined at pickup 2026-07-19 (plan §8, protocol §3). Contingent on the
> **plan §5 amendment #7** (below) being operator-approved and landed BEFORE
> any source (Art. I + amendment rule). Tier-1 from
> `ai_docs/2026-07-19-codex-multi-provider-report.md` §1.

## Context (grounded in source, 2026-07-19)

- `AgentQuery` (`src/pipeline/nodes/build.ts:113`) is a **structural mirror**
  of the Agent SDK's `query` — injected, never imported by the nodes. Its
  `SdkMessage` union (`build.ts:26`) mirrors init `session_id` / assistant
  turns / result-with-usage. `AgentQueryOptions` (`build.ts:76`) carries
  `model`, `resume?`, `agentSpawn?`, `signal`, hooks.
- `src/live-query.ts` is the **ONE** binding of that seam to
  `@anthropic-ai/claude-agent-sdk`'s `query()`. The CLI injects `liveQuery`
  as the `AgentQuery`. **Provider selection = which `AgentQuery` impl the CLI
  injects.**
- The lane's Claude-only default model is `CHORE_MODEL: "sonnet"`
  (`chore.ts`; `build.ts:305` model resolution). Ticket `model:` frontmatter
  overrides it (S6.3).
- `env-policy.ts`'s `SENSITIVE_ENV_KEY` already matches `OPENAI_API_KEY`
  (`_KEY` suffix) — confirm coverage; add any exact-name Codex/OpenAI vars
  that escape the `_TOKEN|_SECRET|_KEY|_PASSWORD|_PASSWD` families.
- Codex CLI surfaces (verified mid-2026, report §3): `codex exec --json`
  emits JSONL (`thread.started`, `turn.started/completed/failed`, `item.*`);
  `codex exec resume <session_id> --json` appends to a prior session. Reference
  impl for the mapping: Vibe Kanban `crates/executors/codex.rs` (report §4).

## Plan §5 amendment #7 (to land, operator-approved; provider-shaped boundary)

> **Draft — awaiting operator approval. Lands in `specs/adw-v1-plan.md` §5
> BEFORE any source (constitution).**

`AgentQuery` (`build.ts`) is a structural mirror of the Claude Agent SDK's
`query`, with `live-query.ts` its single binding. Multi-provider makes
**provider** a dimension **orthogonal to isolation kind**: the same seam gets a
second binding, `src/codex-query.ts`, mapping `codex exec --json` JSONL
(`thread.started` → init `session_id`; `turn.*`/`item.*` → assistant + result
with usage) onto the unchanged `SdkMessage` mirror. **Provider selection is
which binding the CLI injects** — chosen by **target config**
(`provider?: 'claude' | 'codex'`, default `'claude'`) or a CLI `--provider`
flag; **NOT per-ticket frontmatter** (the token-outage case is a whole-run
switch, decision below). Repair **resume** maps `codex exec resume <session_id>`
onto `AgentQueryOptions.resume`. The Claude-only `CHORE_MODEL: 'sonnet'` lane
default becomes a **per-provider default model** (each provider names its own;
ticket `model:` still wins). `AgentQueryOptions` and `SdkMessage` are
**unchanged** — the mirror was already SDK-neutral. **Worktree kind only**:
container/e2b Codex parity (owning the docker-exec / e2b stdio JSONL transport,
re-deriving the m4/m5 teardown hardening, a `~/.codex/auth.json` refresh story)
is **Tier-2, explicitly deferred** (report §1). `env-policy.ts` confirms
Codex/OpenAI credential coverage.

Accepted degradations (report §1): coarser permissioning (Codex `--sandbox
workspace-write` vs Claude's `Bash(bun:*)` allowlist); caps recalibration
(`turns` is Claude-message-scale); SDK-hook capture becomes hooks-based
(adw-m7-02).

## Operator decisions (2026-07-19)

1. **Provider selection is run-scoped, not ticket-scoped.** Target config
   `provider` field (default `claude`) + CLI `--provider` override. Rationale:
   the motivating scenario is a token outage — a whole-run flip, not a
   per-ticket choice.
2. **The `AgentQuery` mirror is the seam — one seam, no new one.** `codex-query.ts`
   is a sibling of `live-query.ts` behind the SAME injected type. The nodes,
   engine, gates, PR lifecycle, journal, and traces are untouched.
3. **Per-provider default model.** Claude → `sonnet` (unchanged); Codex → its
   named default (e.g. `gpt-5.4-codex` — pin the exact id at build). Ticket
   `model:` override still wins (S6.3).
4. **`codex-query.ts` confines the real CLI to a thin edge.** The JSONL→`SdkMessage`
   mapper is a PURE function, unit-tested against captured `codex exec --json`
   fixtures; only a thin process edge (spawn `codex exec`, stream stdout lines)
   touches the child process — the `CiContainerOps` / `E2bOps` precedent.

## Build protocol (red-first, validator-gated per protocol §4)

1. **plan §5 + types (compile-level):** amendment #7 text lands in
   `specs/adw-v1-plan.md` §5; `AgentQuery`/`SdkMessage`/`AgentQueryOptions`
   proven unchanged (existing Claude consumers compile untouched).
2. **`codex-query.ts`:** pure JSONL→`SdkMessage` mapper (fixtures from real
   `codex exec --json` + `resume`), unit-tested; thin process edge injected
   (fake in tests, the `E2bBackgroundImpl` precedent). Resume mapping tested.
3. **model default:** per-provider default model resolution; ticket override
   still wins; Claude default byte-identical.
4. **cli.ts:** `--provider` flag + target-config `provider` field select the
   injected `AgentQuery`; missing Codex auth → descriptive pre-flight refusal
   (the m5-02 preflight precedent); Claude path unchanged when unselected.
5. **env-policy.ts:** confirm/extend the denylist for Codex/OpenAI vars; a
   negative test proves a Codex credential var is scrubbed from agent env.
6. Full suite + lint + tsc; two-axis `/code-review` vs the pre-ticket base;
   **live bar:** a scratch chore completes the full worktree lane under
   `--provider codex` (needs a local `codex` CLI + OpenAI auth — operator-gated).

## Adversarial review — NO-SHIP (2026-07-19)

First build (Sonnet, offline-green 672 tests) **failed adversarial review**
against the real `codex-cli 0.144.4` — full write-up in
`ai_docs/2026-07-19-adw-m7-01-codex-adversarial-review.md`. Root cause: the
builder had no `codex` CLI, so it synthesized the JSONL schema, model id, and
resume argv — all three diverge from the real contract. 5 high + 1 medium:
mapper rejects real JSONL (needs a state reducer), `gpt-5.4-codex` not in the
catalog, resume argv order rejected (`--sandbox` belongs to parent `exec`), CI
repair routes Codex→Claude (+ attempts lack provider/model provenance — see the
gap below), unhandled `AbortError` on cancel (Art. V breach), auth preflight
mis-scoped. **Rework MUST use captured real-CLI fixtures + the live bar as a
hard gate — synthesized fixtures are the failure mode.** Uncommitted diff kept
in-tree as scaffold. Ticket stays NOT done.

## Rework work-order (2026-07-19, operator-approved)

Build against the **captured real-CLI contract** in `test/fixtures/codex/`
(`CONTRACT.md` + `fresh-run.jsonl` + `resume-run.jsonl` + `model-catalog.json`,
codex-cli 0.144.4) — NOT synthesized shapes. Red-first against these fixtures;
the operator-gated live bar remains the final hard gate.

1. **Mapper → pure state reducer** (finding 1): retain `thread_id` (from
   `thread.started` only) + latest `agent_message.text`; parse nested
   `item.{type,text,message}` (incl. `item.type:"error"`); emit the terminal
   result on `turn.completed` (assemble text+usage) / `turn.failed`. Test
   against BOTH captured fixtures.
2. **Default model** (finding 2): pin `gpt-5.4` OR omit `-m` (ride CLI default) —
   NOT `gpt-5.4-codex` (does not exist). Ticket `model:` override still wins.
3. **Resume argv** (finding 3 + the live resume finding): `codex exec resume
   --json -c model=<slug> <session_id> <prompt>` — NO `--sandbox`/`-m` on
   `resume`; model on resume via `-c model=`. Smoke-test against the real parser.
4. **CI-repair provider routing + attempt provenance** (finding 4 — FOLDED IN
   per operator): persist `provider` + resolved `model` on each attempt record;
   thread through ticket parse; `ciRound`/`syncPrState` select the query+auth+
   model matching the ORIGINATING attempt (Codex PR → Codex resume; legacy
   Claude attempt → Claude). Failing-CI test for a Codex-created PR + a
   compat test for a pre-provider Claude attempt. Charge discipline unchanged.
5. **Subprocess lifecycle** (finding 5, Art. V): own `error`+`close`; translate
   expected `AbortError` to clean stream termination; propagate spawn/nonzero
   with stderr context; kill+await child in generator cleanup. No terminal path
   leaves a ticket in-progress.
6. **Auth preflight** (finding 6): injectable preflight via `codex login status`
   under the child's env/`CODEX_HOME` + binary-exists check; cover keyring,
   custom `CODEX_HOME`, missing binary, invalid-auth. Not a bare
   `~/.codex/auth.json` existsSync.

## Review round 2 — remediation (2026-07-19)

The rework passed the round-1 findings but a SECOND `/codex:adversarial-review`
returned NO-SHIP with 5 deeper findings (full write-up:
`ai_docs/2026-07-19-adw-m7-01-codex-adversarial-review-round2.md`). The pattern:
offline-green masked real bugs twice because unit tests inject values at the
seam under test while the real wiring diverged. Fixed (708 tests, tsc+lint):

- **#1 [high] Green path recorded `provider:"claude"`** — `cli.ts` set the
  Codex model in `laneDeps` but never passed `provider`, so `choreLane`'s
  `deps.provider ?? "claude"` fell through and open-pr persisted the wrong
  provider (then failing CI resumed the Codex session through Claude). Fix:
  thread `provider` into `laneDeps`.
- **#3 [high] Resume dropped the sandbox** — resume argv now carries
  `-c sandbox_mode=workspace-write` (matches fresh's `--sandbox`). Real-parser
  smoke test + regression added.
- **#5 [high] Child leaked on abrupt exit** — `iterator.return()` / throw / cap
  skipped the kill+await (it sat after the try/finally). Moved into `finally`
  behind a `normalCompletion` guard; early-return regression test added.
- **#4 [high] Watchdog blind to tool progress** — the reducer emitted only on
  `agent_message`, so a tool-heavy turn (`command_execution` etc.) with no
  agent_message for >10 min was killed as stalled. Now every `item.started` /
  non-agent `item.completed` emits a NON-turn liveness `assistant`
  (`parent_tool_use_id` set) that feeds the watchdog without burning a turn.
  Tested against a REAL tool-using capture (`test/fixtures/codex/tooluse-run.jsonl`).

**Deferred (operator decisions 2026-07-19):**
- **#2 [high] attempt-aware CI auth** → split to **adw-m7-03** (a Claude
  invocation can reach a Codex PR without an auth/binary check before ciRound
  charges; mixed-provider concern).
- **#6 [medium] ticket.model vs attempt.model on resume** → **PARKED** (operator:
  "put aside the in-ticket swap for now"). Current behavior: `ticket.model`
  wins (S6.3). Revisit with adw-m7-03.

## Live bar — PASSED (fresh path, 2026-07-19)

Ran `env -u GITHUB_TOKEN -u GH_TOKEN bun src/cli.ts run --target scratch
--provider codex --ticket m7s-001`. Full worktree lane GREEN end-to-end:
dispatch → provision → build (**real Codex** — read + edited HAIKU.md) → gates
(pass) → commit → push → open-pr → run-end green. **PR #15**
(`silouone/adw-m2-scratch`), ticket → in-review.

- **#1 VALIDATED live:** recorded attempt =
  `{"provider":"codex","model":"gpt-5.4",...}` — NOT claude. The round-2 bug is
  fixed on a real run.
- **#4 held:** the tool-using build completed, no watchdog kill.
- **New live-bar finding (fixed):** the first attempt blocked in `build` on a
  Codex 400 — the run inherited the global `~/.codex/config.toml`
  `model_reasoning_effort = "max"` (a stale value the CLI rejects). Fix:
  `codexExecArgv` now pins `-c model_reasoning_effort=high` on fresh + resume
  (`CODEX_REASONING_EFFORT`), so a factory run is deterministic w.r.t. the
  user's global config without mutating it. Regression test added.

**Residual (not blocking the fresh path):** the **CI-repair resume** path
(`codex exec resume`, #3 sandbox + model threading) is not yet exercised live —
needs a deliberately CI-red PR. Tracked with the mixed-provider CI work in
**adw-m7-03**. The fresh-run lane (this ticket's core) is proven.

## Out of scope

Container/e2b Codex parity (Tier-2); Codex run capture (adw-m7-02); cLens-side
Codex import (the cLens repo, report §2); parallel runs.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; Claude path proven
byte-identical when `--provider` unset; full worktree lane green under
`--provider codex` on a scratch chore (live bar, operator-gated).
