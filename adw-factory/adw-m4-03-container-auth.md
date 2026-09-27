---
id: adw-m4-03-container-auth
type: feat
status: done
priority: 3
created: 2026-07-14
epic: adw-m4
depends: [adw-m4-02-container-workspace]
attempts: []
---
# Container auth — subscription token + git push credentials

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`. Scope widened by refinement: TWO
> credentials cross the container boundary, not one.

## Context

Plan §3 auth + plan §6 risk name the Claude subscription token. The
refinement adds the second credential the coarse ticket missed: the push
node runs `git push` via `workspace.exec` — INSIDE the container — where
the host's keyring credential helper does not exist. Both credentials
follow the same law: env at container start only, never baked into the
image, never written to any file inside it. M2's stale-GITHUB_TOKEN
finding is the cautionary tale for env-token handling (and why every live
factory launch uses `env -u GITHUB_TOKEN -u GH_TOKEN`).

## Deliverables

- Tests first (red): env layering per node, no-file-at-rest checks
- `src/workspace/container.ts` auth slots (`agentEnv()` + exec env
  layering) · whatever start-time env plumbing adw-m4-02's provision needs
- Operator runbook (token mint/refresh/revoke) in this ticket's body

## Requirements

- [x] Agent auth: `CLAUDE_CODE_OAUTH_TOKEN` (from `claude setup-token`)
      enters ONLY as process env at container start / exec time; never in
      the image, never on the container filesystem;
      `ANTHROPIC_API_KEY` stays unset everywhere (N1)
- [x] Push auth: a short-lived token (`gh auth token` on the host) enters
      the same way and is consumed by an in-memory git credential helper
      (inline `credential.helper` on the push command or equivalent) —
      never in `.git/config`, never on disk
- [x] Least exposure: each credential reaches ONLY the exec/agent calls
      that need it (agent nodes: Claude token; push/ci-commit execs: git
      token) — tests pin the exact env each node passes, extending the
      existing agentEnv contract tests
- [x] At-rest proof: after a completed live run, the image scan (m4-01
      procedure) AND `docker diff` + fs grep of the run's kept container
      show no token anywhere — procedure documented + outputs recorded
      (2026-07-17, both live runs — see "Live verification record")
- [x] Runbook: how the operator mints, refreshes and revokes both tokens;
      what expiry looks like at runtime (rate-limit/auth failure must
      land as a graceful blocked run per E4's pattern, not a crash)

## Design as built (2026-07-16; plan §5 amendment #2 operator-approved)

Both credentials are env-only and **exec-time-only** — neither ever enters
the `docker run` environment, so PID-1's env (readable by any in-container
process via `/proc/1/environ`) stays clean; tests pin this.

- **Claude token:** `main()` reads `CLAUDE_CODE_OAUTH_TOKEN` from the launch
  env (container runs only; missing → pre-flight refusal naming
  `claude setup-token`, before any target/SDK load — the stale-token-guard
  precedent). It rides `RunDeps.containerAuth` → provision →
  `workspace.agentEnv()` → the strict-allowlist docker-exec agent path from
  adw-m4-02. It reaches agent execs and NOTHING else (setup runs with
  `ADW_TICKET_ID` only — least exposure; tests pin the exact env).
- **Git push token:** minted lazily ONCE per provision via the injectable
  `gh auth token` edge (`RunDeps.containerAuth.gitToken`, a thunk). Provision
  installs a repo-local `credential.helper` that echoes `$ADW_GIT_TOKEN` —
  the helper is config, the token is not: it expands only at push time from
  the env the push node layers via `Workspace.pushEnv?()` (amendment #2)
  onto EXACTLY the `git push` execs, retries included. The CI round reuses
  the same push node, so its pushes get the same treatment; `ci-commit`
  itself is a local commit and needs no credential.
- **Runtime auth failure (E4):** the SDK reports it as a result message with
  `is_error: true` (spike-observed: "Not logged in" arrives as subtype
  "success" + is_error) — the build/repair stream consumer now fails
  gracefully with the agent's error text, landing the run as blocked instead
  of advancing to doomed gates.

## Operator runbook — tokens

**Claude subscription token (agent auth):**
- *Mint:* `claude setup-token` (interactive browser flow on the host) →
  copy the printed token. Launch container runs with it set:
  `env -u GITHUB_TOKEN -u GH_TOKEN CLAUDE_CODE_OAUTH_TOKEN=<token> bun src/cli.ts run --isolation container …`
- *Refresh:* rerun `claude setup-token` (tokens are long-lived until
  revoked; re-minting supersedes nothing — multiple can coexist).
- *Revoke:* claude.ai → Settings → Sessions/OAuth apps (or
  `claude auth revoke` where available). Factory-side nothing is stored —
  the token lives only in the launch env.
- *Expiry at runtime:* the in-container agent's result arrives
  `is_error: true` ("Not logged in" / "Invalid token") → the run lands
  **blocked** with that reason on the journal (E4); the ticket flips
  blocked, workspace kept. Re-mint and requeue.

**Git push token:**
- *Mint:* automatic — the factory runs `gh auth token` on the host at
  provision time (keyring account `silouone`; short-lived relative to the
  run). Nothing to do beyond having `gh auth login` done once.
- *Refresh:* `gh auth refresh` (or re-login) if gh's stored auth expires.
- *Revoke:* GitHub → Settings → Applications → GitHub CLI (revokes gh's
  grant wholesale).
- *Expiry at runtime:* push fails non-zero → the push node's bounded E6
  retries exhaust → run lands blocked naming the push failure; branch and
  container kept for autopsy.

## Build protocol (Art. I)

1. Red tests: env layering, per-node exposure, no-file-at-rest. Validator
   review per protocol.
2. Implement to green. 3. Live verification (operator go-ahead — real
   tokens): in-container agent call on subscription auth, then the full
   lane.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; LIVE in-container
agent call succeeds on subscription auth; **a toy chore completes the full
lane under `--isolation container`** — this run doubles as the epic exit
criterion (identical pipeline behavior, complete observability artifacts,
agent spans parented under the run — S5.3); at-rest scan clean.

## Live verification record (2026-07-17, operator-approved series)

1. **Live in-container agent probe** — one real SDK `query()` through the
   PRODUCTION seam (`liveQuery` + `agentSpawn`) against a fresh
   `adw-agent:2.1.210` container, `CLAUDE_CODE_OAUTH_TOKEN` via options env:
   init message with in-container cwd, result `LIVE-PROBE-OK`,
   `is_error:false`, stream ended cleanly. Subscription auth works
   in-container (N1).
2. **Full-lane live run #1 — `m4s-001`** (scratch target,
   `--isolation container`, real Sonnet agent + real push): green end-to-end
   — dispatch → provision → assemble-prompt → build (37 turns in-container)
   → gates → commit → push (credential helper + `ADW_GIT_TOKEN`) → open-pr.
   PR https://github.com/silouone/adw-m2-scratch/pull/6, ticket in-review,
   container kept. Journal complete (8 nodes); 9 spans, all parented under
   the run span, build typed AGENT (S5.3). **Live finding:** capture
   `ok:false` — clens-hook resolves its sink from the payload's
   in-container `cwd` (m3-06 F7) and `transcript_path` is host-unreadable;
   the m6-04 completeness check caught and journaled it exactly as designed.
3. **Capture-parity fix** (red-first, validator-gated): hook-payload rewrite
   for container runs — `cwd` → run dir, `transcript_path` → docker-cp'd
   host copy (`liveContainerTranscriptFetch`, 0700 staging under
   `<runDir>/.clens/transcripts/`). Worktree payloads byte-identical.
4. **Full-lane live run #2 — `m4s-002`** (the epic-exit run): green,
   PR https://github.com/silouone/adw-m2-scratch/pull/7; **capture
   `ok:true`**, session file present (37 records, every one stamped
   `adw_ticket_id: m4s-002`), no end-of-run downgrade. Complete
   observability artifacts: journal + spans + capture (S5.3).
5. **At-rest scan, both kept containers** (m4-01 procedure + `docker diff` +
   fs grep for BOTH actual token values): zero hits anywhere on the
   container filesystems; PID-1 env carries zero token-shaped vars
   (exec-time-only proven); repo-local `credential.helper` stored
   UNEXPANDED (`$ADW_GIT_TOKEN` literal); host run dirs
   (sessions/transcripts staging) also token-free. Image unchanged from the
   m4-01 clean scan.

## M4 milestone review absorption (2026-07-17, two-axis /code-review)

- **Fixed red-first (spec c2, real):** both docker-exec edges built
  `-e KEY=<value>` argv — credentials were `ps`-visible on the host during
  each exec. Now ONE shared pure `dockerEnvFlags` emits bare `-e KEY` flags
  (values ride the docker CLIENT's process env; verified live) at both the
  agent-spawn and workspace-exec edges. This also resolved the review's
  duplicated-flatMap note.
- **Deviation recorded (spec c1):** the requirement's literal "never in
  `.git/config`" is NOT met for the HELPER — provision persists the
  credential.helper (which stores only the literal string `$ADW_GIT_TOKEN`,
  no credential) in the clone's repo-local config. A truly config-free
  alternative (`GIT_CONFIG_*` env config) needs git ≥ 2.31; the image ships
  2.30.2. The security property the line was protecting — the TOKEN never at
  rest — holds and is live-proven (at-rest scans). Requirement text stands
  as written; this note is the honest delta.
- **Noted (spec c3):** a LOCAL-path origin is bind-mounted rw at
  `/adw/origin` — writable by the agent, an S4.2 softening that exists only
  in the fixture/local case (real targets have URL origins, no mount).
- **Follow-up filed:** `adw-m4-04-ci-round-container` (spec a1 — the CI
  mini-lane is worktree-only; previously documented but untracked).

## Out of scope

API-key billing (spec non-goal); remote/E2B auth (adw-m5); credential
rotation automation.
