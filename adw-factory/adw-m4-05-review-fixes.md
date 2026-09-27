---
id: adw-m4-05-review-fixes
type: feat
status: done
priority: 3
created: 2026-07-17
epic: adw-m4
depends: [adw-m4-03-container-auth]
attempts: []
---
# M4 Codex adversarial-review absorption

External adversarial review (Codex, 2026-07-17, diff 31baae0..HEAD; the
accepted milestone external review per the build protocol). No Critical; 4
High, 4 Medium. Each challenged against code + LIVE probes. Operator scope
decision 2026-07-17: **fix the core-3 credential-containment findings now
(red-first, validator-gated); track the rest.**

## Fixing now (core-3)

- **High-2 — OAuth token theft via `claude` swap (CONFIRMED live).**
  `~/.local/bin/claude` is writable by the `bun` user the agent runs as
  (`touch $(command -v claude)` succeeds); the agent has `Bash(bun:*)` =
  arbitrary JS. A malicious/injected setup or build agent replaces the
  binary; the next/repair spawn hands the replacement
  `CLAUDE_CODE_OAUTH_TOKEN`. Fix: install `claude` root-owned in a
  bun-non-writable dir on PATH, and drop the `~/.local/bin` PATH prepend so
  it cannot be shadowed. (containers/Dockerfile.)
- **High-1 — push-token confused deputy (CONFIRMED live).** The credential
  helper returns the token for ANY host (`host=attacker.invalid` probe
  returned the sentinel), and it is global; an agent that rewrites `origin`
  redirects the factory's push token to itself. Fix: scope the helper to the
  real origin's host (`credential.<scheme://host>.helper`), so a redirected
  origin gets no credential. The orthogonal "`gh auth token` is the
  operator's wholesale default-host credential, not a per-run scoped token"
  is a real `gh` limitation — documented, not fixed here. (container.ts.)
- **Medium-3 — OAuth token in `clens-hook` env (CONFIRMED live).**
  `liveClensSpawn` forwards the full `process.env`; the hook needs only
  `ADW_TICKET_ID`. Fix: scrub the inherited env through the existing
  `sanitizeAgentEnv` denylist (strips `*_TOKEN`, incl.
  `CLAUDE_CODE_OAUTH_TOKEN`). (clens.ts.)

## Tracked, not fixed here (with rationale)

- **High-3 — no hard OS kill; crashed container unreclaimable (Art. V).**
  Real: the deadline signal is polled between stream yields, never passed as
  the SDK `abortController`, and the custom spawn drops
  `SpawnOptions.signal`; `docker exec` client-kill leaves the in-container
  process alive (documented in container.ts). Separately, the attempt is
  recorded only AFTER the lane, so a crash-stranded container is invisible
  to `adw clean` (which scans `attempts[]`). Engine-level work → **filed
  adw-m4-06-hard-stop-container**. Note it is partly pre-existing (the
  never-yielding-stream case predates M4, a documented M6 watch item); M4 is
  where a real OS kill first BECOMES possible.
- **High-4 — rw `/adw/origin` mount breaches S4.2.** Real S4.2 breach (a
  write path outside `/work/repo`, agent-reachable via `bun`), but
  **local-path-origin ONLY** — a test-fixture construct. Production N4
  targets use URL origins, where `originIsLocal` is false and NO mount is
  created. Accepted as test-harness scope; documented. If container
  isolation ever needs a local-origin production target, revisit (push from
  host instead of a rw mount). Tracked in adw-m4-06.
- **Medium-1 — CI mini-lane worktree-only.** Already filed
  **adw-m4-04-ci-round-container**. The shipped behaviour fails safe (a
  container PR going red blocks with "state error, no round charged"),
  it does not silently mis-push.
- **Medium-2 — transcript-cp failure doesn't downgrade capture health.**
  Core is the known m6-04 durable-write limitation (a dropped whole event
  is undetectable without an expected count). A fetch-failure-on-Stop
  downgrade is a small defensive enhancement → adw-m4-06.
- **Medium-4 — origin URL unescaped / SSH origins break push.** The
  SSH-origin functional gap (HTTP helper can't auth ssh://) and
  credential-in-URL argv/`.git/config` exposure are real for
  operator-supplied origins → adw-m4-06.

## Build protocol (Art. I)

Red tests first (docker-gated for the image + helper facts; pure unit for
the clens env scrub), validator-gated, then implement to green + rebuild the
image. Live re-verify a container run's capture + push still work.

## Verify

`bun run lint && bunx tsc --noEmit && bun test` green; the three red tests
pass against the rebuilt image; a live container run still completes the
full lane with capture ok:true and push landing on the real origin.

## Live re-verification (2026-07-17, m4s-003 on the rebuilt image)

Green full lane, PR https://github.com/silouone/adw-m2-scratch/pull/8,
capture **ok:true** (56-record session). In the actual run container
`adw-m4s-003-…`, as the `bun` agent user:

- **High-2:** `command -v claude` → `/usr/local/bin/claude` (root-owned);
  `rm` of it → `Permission denied` (rc=1). The agent cannot swap the binary.
- **High-1:** only `credential.https://github.com.helper` is set (the scratch
  origin's host); the global `credential.helper` key is empty. A redirected
  origin would get no token; push to the real github.com origin worked.
- At-rest: fs grep for BOTH live token values across the whole container →
  zero hits; PID-1 `/proc/1/environ` → zero token-shaped vars.

445 tests green; rebuilt-image secret scan clean (Config.Env has no
`~/.local/bin` PATH entry either — the shadow surface is gone).
