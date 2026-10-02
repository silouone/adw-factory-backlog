---
id: adw-web-01-a-stable-address-d7064f
type: feat
status: in-progress
priority: 2
created: 2026-10-01
depends: []
attempts: [{"runId":"adw-web-01-a-stable-address-d7064f-1790883163351","branch":"adw/adw-web-01-a-stable-address-d7064f","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-web-01-a-stable-address-d7064f-1790883163351/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"},{"runId":"adw-web-01-a-stable-address-d7064f-1790891798135","branch":"adw/adw-web-01-a-stable-address-d7064f-2","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-web-01-a-stable-address-d7064f-1790891798135/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# adw web can keep the same address across restarts

> **Re-queued 2026-10-02:** the second attempt (`…-1790891798135`) used up 3 repair rounds on failures in `adw run` CLI tests (green path, codex, container). The diff does edit `src/cli.ts`, but `test/cli.test.ts` run alone on that workspace passes 49/49. The failures were load: web-01 and web-02 started within a second of each other. If `test/cli.test.ts` goes red in your run, treat it as real, not flaky. Its workspace diff (justfile, `src/cli.ts`, the v1.2 amendment, `test/cli.web-launch.test.ts`) is still on disk if a fresh run wants a reference.

> **Re-queued 2026-10-01:** the first attempt was refused at `baseline-green-check` on `6b058da`, but the base was not red. The passes failed on different (flaky) rebase tests, and the rule from adw-bug-31 cached that as red. See adw-bug-35 and adw-bug-36. The cached verdict was removed.

`adw web` mints a random token on every launch, and takes its port from a flag. A tool that
embeds or reads it, the HQ (spec `~/personal_project/silou-hq/docs/spec-v1.md`, "the factory contract"), must be handed a fresh
`?token=` URL after every factory restart. Today that's a manual copy.

## What to build

- `adw web` reads an optional token and port from the environment (e.g. `ADW_WEB_TOKEN`, `ADW_WEB_PORT`). The flag beats the environment; without either, behaviour is byte-identical to today (random token, current port default).
- The token is still required on every request; loopback-only binding and the read-only server are unchanged.
- **Spec amendment first** (Amendment rule): `adw-v1.2-live-view.md` describes a per-launch token. Amend it to say "per-launch by default, operator-fixable".

## Red first (Art. I)

- With the env token set, a request carrying it is served, and one without it is refused.
- With nothing set, two launches mint different tokens (today's behaviour, pinned).
- A grep-tested guard still finds no mutating route.

## Acceptance criteria

- [ ] `ADW_WEB_TOKEN=x ADW_WEB_PORT=7399 just web` serves `http://127.0.0.1:7399/?token=x`, and survives a restart with the same URL.
- [ ] Default behaviour is unchanged when neither is set.
- [ ] `bun run lint && bunx tsc --noEmit && bun run test` green.

## Blocked by

- (nothing)
