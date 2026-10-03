---
id: hq-40-the-claude-runtime-maps-the-stream-f39531
type: feat
status: in-review
priority: 2
depends: []
created: 2026-10-03
caps: {minutes: 150, turns: 400}
attempts: [{"runId":"hq-40-the-claude-runtime-maps-the-stream-f39531-1791039668555","branch":"adw/hq-40-the-claude-runtime-maps-the-stream-f39531","workspace":"/Users/silouane/personal_project/adw-factory/runs/hq-40-the-claude-runtime-maps-the-stream-f39531-1791039668555/workspace","outcome":"in-review","pr":"https://github.com/silouone/silou-hq/pull/40","provider":"claude","model":"claude-sonnet-5-5","rebased":"39b62fef6eb9be7b56d49e1a155184190f09f216"}]
---
# The companion's Claude runtime: config, SDK options and the message stream, pure and tested

> Spec: `~/personal_project/silou-hq/docs/spec-v4-companion.md` (binding; it amends rule #1 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` (incl. amendment v2.1) + `docs/spec-v3-mail-calendar.md` still bind otherwise). Rules: `CLAUDE.md`. Claude-specific code lives ONLY in `src/agent/runtime/claude.ts`. No test spawns the real `claude` binary.

## What to build

Story 18, plus the runtime half of stories 1–4 and 6 (spec-v4 → Config; Server modules: `types.ts`, `runtime/claude.ts`, `companion-prompt.md`):
- **`agent` in `hq.config.json`,** parsed fail-fast, naming the field: an unknown runtime; a `bin`, `cwd` or `path` entry that isn't absolute after `~` expansion. Absent → no agent.
- **`src/agent/types.ts`:** `AgentEvent`, `Runtime`, `ToolCall`, exactly as the spec gives them.
- **Add `@anthropic-ai/claude-agent-sdk` pinned to `0.3.287`** (the probed version).
- **Pure `claudeOptions(config, q)`:**
  - the system binary (`pathToClaudeCodeExecutable`), never the SDK's bundled one;
  - `cwd`, `permissionMode: "bypassPermissions"`, `allowDangerouslySkipPermissions`, `settingSources: ["user"]`;
  - `extraArgs.settings` forcing `{"outputStyle":"default","disableAllHooks":true}`;
  - `includePartialMessages`, `resume` only when given, `model` only when non-null;
  - `env` with the configured `PATH`;
  - `appendSystemPrompt` from `src/agent/companion-prompt.md`.
- **Pure `mapMessages`** (SDK messages → `AgentEvent`s):
  - `markdown` comes **only** from `stream_event` `text_delta`s, cumulative, with text blocks after a tool call joined by `\n\n`;
  - `activity` comes **only** from `assistant` `tool_use` blocks (their text blocks are ignored), one line per tool use (`<Tool> · <command|path|pattern>`, ≤ 120 chars);
  - `init` and `result` become `session`; `result` becomes `done` (`is_error` → `ok: false`, error = the result text), with `total_cost_usd`.
- **The edge, `createClaudeRuntime(config)`:**
  - `query()` with a `PreToolUse` callback (timeout 600 s) that awaits `gate(call)` and returns `permissionDecision: "deny"` with the reason on refusal;
  - `signal` wired to the SDK's `abortController`;
  - a spawn failure (binary missing, not logged in) yields `done` with `ok: false` and an error naming the binary path. No throw escapes.
- **`src/agent/companion-prompt.md`** with the four points the spec lists.
- **A runtime registry** `{ claude: createClaudeRuntime }`, keyed by `agent.runtime`.

## Red first

- **Config parse:** valid, absent, unknown runtime, relative path. Each error names the field.
- **`claudeOptions` snapshot:** forced output style and disabled hooks, bypass mode, explicit `PATH`, `resume` and `model` present only when set.
- **`mapMessages` over `test/fixtures/agent/claude-stream.jsonl`** (real capture):
  - markdown is cumulative, is `"pong"` before the tool call, then `"pong\n\n…"`, and is never doubled by the `assistant` messages;
  - exactly one `activity` for the one `tool_use` (`Bash`);
  - `session` equals `13f13874-944e-4f2b-b80f-36ee1f8bc084`;
  - `done.ok` is true, with a cost.
- **A hand-made `is_error` fixture** gives `done.ok: false` carrying the error text.

## Acceptance criteria

- [ ] Nothing outside `src/agent/runtime/claude.ts` imports the SDK or names Claude.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
