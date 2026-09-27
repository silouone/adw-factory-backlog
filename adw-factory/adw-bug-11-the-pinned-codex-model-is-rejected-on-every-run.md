---
id: adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run
type: bug
status: done
priority: 1
created: 2026-09-17
depends: []
attempts: [{"runId":"adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run-1789673356704","branch":"adw/adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-bug-11-the-pinned-codex-model-is-rejected-on-every-run-1789673356704/workspace","outcome":"blocked","provider":"claude","model":"sonnet"}]
---
# Every `--provider codex` run dies at its first agent node, because the pinned model slug does not exist

> The second provider has been unusable since it shipped. `CODEX_DEFAULT_MODEL`
> names a slug the Codex CLI rejects, so the first agent node of every Codex run
> fails before it does any work.

## Context

`src/codex-query.ts:69`:

```ts
export const CODEX_DEFAULT_MODEL = "gpt-5.4";
```

Operator-verified (2026-07-19 probe, re-verified 2026-09-17 — the constant is
unchanged): `gpt-5.4` is **rejected** on a ChatGPT-account login. Every
`--provider codex` run therefore fails at its first agent node unless the
ticket carries a `model:` override.

The verified working slug is **`gpt-5.6-sol`** — also the operator's own
`~/.codex/config.toml` default — confirmed through the factory's full composed
argv:

```
exec --json --sandbox workspace-write -m gpt-5.6-sol \
  -c model_reasoning_effort=high --dangerously-bypass-hook-trust <hooks> -
```

**The pin itself is correct and must stay.** Its purpose (decision 3, finding 4)
is that the resolved model is persisted on the attempt record and re-threaded
through `-c model=` on a CI-repair resume — a run needs to know its own default
in order to persist it. The pin is right; the value is wrong.

## Why priority 1

`specs/adw-v1.8-agent-profiles.md` R5 routes individual stages to Codex. Every
one of those routes lands on this constant. The profile work cannot be
end-to-end tested while the provider is dead on arrival.

## Requirements

- [ ] `CODEX_DEFAULT_MODEL` is a slug the Codex CLI actually accepts. Re-probe
      before trusting any literal — OpenAI rotates these, and a slug that
      worked in a prior probe is not evidence it works today.
- [ ] The doc comment records **when** the slug was last probed, so the next
      failure is diagnosable as rotation rather than re-derived from scratch.
- [ ] A test pins the constant against the committed model catalog fixture
      (`test/fixtures/codex/model-catalog.json`) — the same fixture that caught
      the first pass's non-existent `gpt-5.4-codex`. A slug absent from the
      catalog must fail the suite, not a live run.
- [ ] No behaviour change beyond the value and its guard: the pin, its
      persistence on the attempt record, and the `-c model=` resume path are
      untouched.

## Verify

- [ ] Red first: a test asserting `CODEX_DEFAULT_MODEL` is present in the model
      catalog fixture goes RED on `gpt-5.4`, green after the fix.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.
- [ ] Operator-executed (not agent-executed, real spend): one live
      `--provider codex` run on a scratch target reaching at least its first
      agent node without a model rejection.

## Out of scope

Per-stage provider routing (`adw-profile-02`); Codex container/remote parity;
changing `CODEX_REASONING_EFFORT`.
