---
id: adw-m7-02-codex-capture
type: feat
status: done
priority: 3
created: 2026-07-19
epic: adw-m7
depends: [adw-m7-01-codex-worktree]
attempts: []
---
# cLens capture for Codex runs

> Coarse ticket — **refine requirements at pickup** (plan §8). Report §1 step 5
> + §3 (`ai_docs/2026-07-19-codex-multi-provider-report.md`).

## Scope (coarse)

A `makeCodexClensHooks` analog of `src/observability/clens.ts`: Codex runs on
the worktree lane (adw-m7-01) are captured to the run-dir cLens sink via Codex's
native hooks (`.codex/hooks.json` / `config.toml [hooks]` — near-isomorphic to
Claude Code hooks, report §3), stamped with `adw_ticket_id`, journalled, behind
the same circuit breaker. Guiding split (report intro): **hooks = live, stamped,
per-tool-call capture → the factory's need.**

## Verification gate (do first, report §3)

30-min spike: wire the existing `clens-hook` into `.codex/hooks.json` in a
scratch project, run one Codex session, inspect what lands in
`.clens/sessions/` — confirms tool-payload field names
(`tool_name`/`tool_input`/`tool_response`?) and how far "capture is nearly free"
really goes. Findings drive the refinement.

## Out of scope

The provider swap itself (adw-m7-01); cLens-side Codex session-file import (the
cLens repo, report §2).

## Refinement — 2026-07-20 (spike-driven; awaiting operator approval per protocol §3)

### Spike findings (ground truth — scratch `codex exec` run, gpt-5.4)
- **Hooks FIRE under `codex exec`** (the doc is silent on non-interactive — proven
  empirically). Capture is **near-free**: the existing `clens-hook` captured a
  codex session UNCHANGED — all 5 wired events (`SessionStart`,
  `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`) landed in a valid
  `<sid>.jsonl` with rich tool payloads (`tool_name`, `tool_input`,
  `tool_use_id`, `turn_id`, `transcript_path`→ the codex rollout file,
  `permission_mode`).
- **`sid` = codex thread_id = the factory's journaled `sessionId`** — the m3-05
  capture↔run join key works for Codex (same as Claude).
- Codex resolves `.codex/hooks.json` from the **workspace cwd** (confirmed firing
  from there); git sees it as `?? .codex/` — i.e. INSIDE the workspace, unlike
  Claude's `.clens` sink which sits above it.
- Payload has NO `adw_ticket_id` unless `ADW_TICKET_ID` is in the hook child env.
- Trust: needs `--dangerously-bypass-hook-trust` (our clens-hook is the only,
  vetted hook — the flag's documented automation use).

### Requirements
- **R1** — For a codex worktree run, provision `.codex/hooks.json` in the
  workspace, each codex event → `clens-hook <event>` (codex event set:
  `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`,
  `SubagentStart`, `SubagentStop`, `PermissionRequest`; codex-specific — NO
  `SessionEnd`/`PostToolUseFailure`/`Notification`; has `Pre/PostCompact`).
- **R2** — Pre-seed `.clens/sessions/` in the runDir (ABOVE the workspace),
  reusing `clens.ts`'s sink-path helpers, so no `.clens` reaches the workspace/PR
  (Claude parity).
- **R3** — `codexExecArgv` (fresh + resume) carries `--dangerously-bypass-hook-trust`
  (unit-pinned, Part H). **[design choice A]**
- **R4** — `ADW_TICKET_ID` set in the codex child env (`codexChildEnv`) so
  clens-hook stamps `adw_ticket_id` into every payload.
- **R5** — Add `.codex` to the commit + ci-commit sweep exclusion (the `.claude`
  precedent) — the workspace `.codex/hooks.json` must never reach a PR.
  **[design choice B]**
- **R6** — Journal a `capture` record + completeness check (reuse
  `checkCaptureCompleteness` + circuit breaker); codex runs journal
  `capture ok:true/false` like Claude/E2B.
- **R7** — Wire into cli.ts's codex run path; the Claude path stays
  byte-identical.

### Design note
`makeCodexClensHooks` is NOT an in-process `AgentHooks` bag (codex is a
subprocess — no SDK hook callback). It's a **workspace/argv/env provisioner**:
writes the config, pre-seeds the sink, augments argv+env. The Claude
`makeClensHooks` (SDK path) is untouched. Shared: sink-path helpers + env-policy
+ the `CLENS_HOOK_COMMAND` constant.

### Verify
`bun run lint && bunx tsc --noEmit && bun test` green; unit tests that a codex
run provisions `.codex/hooks.json`, pre-seeds `.clens` above the workspace, the
argv carries the bypass flag, `.codex` is swept-excluded, and capture is
journalled. **LIVE bar (billed):** a `--provider codex` scratch run captures a
valid `<sid>.jsonl` with `adw_ticket_id` stamped, and the PR contains NO
`.codex/` or `.clens/`.

## DELIVERED — 2026-07-20 (live-proven; TWO live-bar rewires supersede R1/R4/R5 above)

The refinement's file-based design was NO-SHIP'd by the live bar TWICE — the
delivered design is simpler. Final: capture rides the ARGV, no workspace file.

- **R1 (superseded):** the hooks config is carried via `-c hooks.<Event>=[...]`
  flags on the codex argv (`codexHookConfigArgs`, spliced by `codexExecArgv` on
  fresh + resume), NOT a workspace `.codex/hooks.json` file. **Live-bar root
  cause:** codex does NOT load a workspace `.codex/hooks.json` when cwd is a git
  WORKTREE — the worktree's `.git` is a FILE, and codex's project-root discovery
  walks up for a `.git` DIRECTORY, overshooting to the outer repo (no `.codex/`).
  The clens-hook command is resolved to an ABSOLUTE path (`resolveOnPath`) —
  codex runs hooks with a reduced PATH, so a bare command never resolves.
- **R2 (met):** `.clens` sink pre-seeded in the runDir via the unconditional
  `makeClensHooks` call in cli.ts (`seedClensSink` side-effect, both providers).
- **R3 (met):** `--dangerously-bypass-hook-trust` on fresh + resume argv.
- **R4 (met, different mechanism):** `adw_ticket_id` is stamped POST-HOC at run
  end — `checkCodexCapture(runDir, sid, ticketId, append)` rewrites the captured
  file, adding `adw_ticket_id` into each record's `data` (Claude parity,
  `clens.ts:361`). **Root cause:** the Claude stamp is injected by the factory's
  OWN forwarder into the payload; codex bypasses that path (pipes its payload
  straight to clens-hook, which does NOT read `ADW_TICKET_ID` from env), so the
  env approach cannot work.
- **R5 (DROPPED):** no `.codex` sweep exclusion — nothing is written to the
  workspace, so `chore.ts`/`ci-round.ts`/`push.ts` stay `.claude`-only baseline.
- **R6/R7 (met):** capture journalled `ok:true/false`; Claude path byte-identical.

**Live bar (m7s-006, PR #20):** `capture ok:true`, all 5 event types, every
record stamped `data.adw_ticket_id`, `sid` = journaled sessionId, PR clean.

### DEFERRED — codex CI-repair-round capture (R6 for the resume case)
A codex ticket that enters a CI-repair round: the `-c hooks` args ride resume
too, so hooks DO fire and append to the SAME `<sid>.jsonl` — but `ci-round.ts`
never calls `checkCodexCapture`, so the ci-round-appended records are NOT
`adw_ticket_id`-stamped (partial stamp) and the round gets no positive `ok:true`
verdict (only the negative-only `checkCaptureCompleteness`). Fresh-dispatch — the
ticket's stated verify + live-bar scope — is fully delivered. **Follow-up
candidate:** call `checkCodexCapture(dirname(attempt.workspace), sid, ticketId,
append)` in the codex ci-round branch (idempotent re-stamp of the shared file +
an ok:true verdict). Tracked here, not silently dropped (m6-04 lesson).
