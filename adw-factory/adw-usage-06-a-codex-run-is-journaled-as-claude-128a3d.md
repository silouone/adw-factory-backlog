---
attempts: [{"runId":"adw-usage-06-a-codex-run-is-journaled-as-claude-128a3d-1790553521322","branch":"adw/adw-usage-06-a-codex-run-is-journaled-as-claude-128a3d","workspace":"/Users/silouane/personal_project/adw-factory/runs/adw-usage-06-a-codex-run-is-journaled-as-claude-128a3d-1790553521322/workspace","outcome":"in-review","pr":"https://github.com/silouone/adw-factory/pull/140","provider":"claude","model":"sonnet"}]
id: adw-usage-06-a-codex-run-is-journaled-as-claude-128a3d
type: bug
status: in-review
priority: 1
created: 2026-09-28
caps: {minutes: 60, turns: 200, stallMinutes: 20}
depends: []
---
# A Codex run is journaled as `provider: "claude"`

> **Found 2026-09-28 during the cross-provider cost audit.**
> `codexPlaceholderProfiles` (`src/pipeline/lanes/shared.ts:683`) builds the
> synthetic `(codex-default)` profile with `provider: "claude"`. Every
> Codex run's `node-end` usage therefore carries
> `profile.provider: "claude"` (seen on
> `runs/cqc-fe-01-drawer-keeps-keyboard-focus-1790524928011/journal.jsonl`).
> Any spend-by-provider rollup would file Codex spend under Claude.

## Requirements

- [ ] **R1 — red first.** A unit test: every stage profile returned by
      `codexPlaceholderProfiles("gpt-5.6-sol")` has `provider === "codex"`.
      It fails on `main`.
- [ ] **R2 — fix.** The placeholder profile says `provider: "codex"`. If
      `AgentProfile["provider"]` does not admit `"codex"`, widen the type.
      Do not add a cast.
- [ ] **R3 — no Claude-path change.** Existing profile-registry tests stay
      green, unchanged.

## Verify

`bun run lint && bunx tsc --noEmit && bun run test`.
