---
id: adw-model-01-deep-runs-opus-5-5
type: chore
status: queued
priority: 2
created: 2026-09-28
depends: []
attempts: []
---
# The `deep` profile still runs Opus 4.8: Opus 5.5 needs Claude Code ≥ 2.1.280

## The problem

`PROFILE_REGISTRY.deep` sends the bare alias `opus`. The factory's agent
binaries are old:

- worktree runs: SDK-bundled CLI 2.1.209 (`@anthropic-ai/claude-agent-sdk@0.3.209`)
- container and E2B runs: image CLI 2.1.210

On those binaries `opus` resolves to `claude-opus-4-8`. Pinning
`claude-opus-5-5` fails with an API 400, probed live on 2026-09-28:

> Claude Code 2.1.209 does not support this model; version 2.1.280 or newer is required.

`standard` and `swift` are already pinned to `claude-sonnet-5-5`, which runs
cleanly on 2.1.209.

## Done when

1. `agent-sdk` is bumped to a version bundling CLI ≥ 2.1.280 (0.3.284 bundles 2.1.284).
2. The container image and E2B template are rebuilt on the same CLI version:
   - `containers/Dockerfile` ARG
   - `ADW_AGENT_IMAGE`
   - `E2B_TEMPLATE_CLAUDE_VERSION`
   - `IMMUTABLE_CLAUDE`, re-probed live because its existence is a live fact (m5-02 lesson)
3. `deep.model` is `claude-opus-5-5`, test-first in `test/pipeline/profiles.test.ts`.
4. One live dispatch on each isolation kind shows `claude-opus-5-5` in the
   journal's `system/init` with `is_error:false`.

## Risks to re-check

- A previous SDK auto-bump (0.3.214) broke container-style spawn (cwd host-check).
- The m5-05 settlement bound was reasoned against 0.3.209's teardown source.
- Probe with `--setting-sources project`. A user-level `advisorModel` makes
  the API reject a Sonnet 5.5 request whose advisor is Opus 4.8, and that
  error has nothing to do with the model under test.
