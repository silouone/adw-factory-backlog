---
id: adw-m4-01-container-image
type: chore
status: done
priority: 3
created: 2026-07-14
epic: adw-m4
depends: [adw-m3, adw-m6]
attempts: []
---
# Prebaked container image

> Refined at pickup 2026-07-15 (plan §8, protocol §3). Operator approval
> required before `in-progress`. `depends` now includes adw-m6 — the
> 2026-07-14 reorder (shakedown before isolation expansion) made binding.

## Context

The container workspace kind (S4.5) runs the agent INSIDE the container —
that is what "fully contained local execution" means, and why the image
prebakes the Claude Code CLI alongside bun + git (plan §3). The image is
built from a committed Dockerfile so it is a reviewable, reproducible
artifact (N5 spirit). No credentials of any kind are baked in (plan §6,
N1) — auth arrives only as env at container start (adw-m4-03).

## Deliverables

- `containers/Dockerfile` (committed, factory repo) + a header documenting
  the build command, tag convention, and pinned versions
- Optional thin build script only if the docker invocation needs >1 flag
  (Art. II — no speculative tooling)

## Requirements

- [ ] Base `oven/bun:<pinned>`; adds `git` and the Claude Code CLI
      (`@anthropic-ai/claude-code`, version pinned via build `ARG`); a
      non-root user; a fixed workdir (e.g. `/work`) that adw-m4-02 will
      treat as the workspace root. `gh` is NOT installed — open-pr and all
      PR reads run on the host (Art. IV surface stays host-side)
- [ ] Reproducible: every version pinned (base tag, claude-code ARG); tag
      convention `adw-agent:<claude-code-version>` documented in the
      Dockerfile header
- [ ] Zero secrets at rest: no ANTHROPIC_API_KEY, no OAuth token, no
      gitconfig credentials, no copied host dotfiles (N1, plan §6); the
      scan procedure (`docker history --no-trunc` + fs grep for token
      patterns) is documented in the Dockerfile header and run
- [ ] Works under the operator's runtime (OrbStack; docker-CLI
      compatible) — build + smoke run verified there

## Build protocol

Chore (no source modules): write the Dockerfile, build, run the three
version smoke checks + the secret scan, record outputs in this body.
TDD Art. I applies to any TS helper this ticket adds (none expected).

## Verify

`docker build` succeeds; `bun --version`, `git --version`,
`claude --version` all work in a container from the image; secret scan
clean; `bun run lint && bunx tsc --noEmit && bun test` untouched-green.

## Build record (2026-07-16)

Image built on the operator's OrbStack runtime (docker server 29.4.0):

```
docker build -f containers/Dockerfile \
  --build-arg CLAUDE_CODE_VERSION=2.1.210 \
  -t adw-agent:2.1.210 containers/
# → sha256:071918077553c01a98c08a32268811ef71e9263e0c60b2b0e585796d2b0185cb
```

Pins: base `oven/bun:1.2.4` (matches host bun), claude-code `2.1.210` via
the native standalone installer (`claude.ai/install.sh <version>` — no npm,
no Node runtime in the image), matching the host's known-working CLI.

Smoke checks (in a container from the image):

```
bun: 1.2.4
git: git version 2.30.2
claude: 2.1.210 (Claude Code)
user: bun uid=1000        # non-root (base image's `bun` user)
pwd: /work                # fixed workspace root for adw-m4-02
```

Secret scan (procedure in the Dockerfile header) — all three steps clean:

1. `docker history --no-trunc` — 0 credential-shaped layer commands.
2. `docker inspect .Config.Env` — only PATH/HOME/bun-runtime vars, no
   `*_TOKEN`/`*_KEY`/`*_SECRET`:
   `["PATH=…","BUN_RUNTIME_TRANSPILER_CACHE_PATH=0","BUN_INSTALL_BIN=/usr/local/bin","HOME=/home/bun"]`
3. fs grep for `sk-ant-|ANTHROPIC_API_KEY=|ghp_|github_pat_` over
   `/home /root /work /etc /usr/local` — no hits.

Validator: APPROVE (every checkbox re-verified against live docker state,
incl. `gh` absent and installed `claude --version` == the ARG). One
low-severity observation absorbed into the Dockerfile header: install.sh
content is fetched unpinned, so the build is version-reproducible but not
byte-for-byte reproducible — acceptable at this ticket's bar.

M4 milestone review note (2026-07-17): the requirement named the npm
package `@anthropic-ai/claude-code`; the image installs the NATIVE
standalone binary (same product, same pinned version, no Node runtime) via
the official installer — a deliberate mechanism substitution that was
documented in the build record but not called out as a delta until this
note.

No build script added: the docker invocation is a single documented command
(Art. II — the optional script's >1-flag threshold reads as "needs
orchestration", which it doesn't). `gh` confirmed absent (never installed).

## Out of scope

Workspace implementation (adw-m4-02); any auth (adw-m4-03); remote/E2B
(adw-m5).
