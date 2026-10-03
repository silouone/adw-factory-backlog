---
id: hq-41-deletions-and-secrets-wait-for-the-operator-a263ea
type: feat
status: in-progress
priority: 2
depends: []
created: 2026-10-03
caps: {minutes: 120, turns: 300}
attempts: []
---
# The approval classifier: deletions and secret reads are flagged, everything else passes

> Spec: `~/personal_project/silou-hq/docs/spec-v4-companion.md` (binding; it amends rule #1 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` (incl. amendment v2.1) + `docs/spec-v3-mail-calendar.md` still bind otherwise). Rules: `CLAUDE.md`. Claude-specific code lives ONLY in `src/agent/runtime/claude.ts`. No test spawns the real `claude` binary.

## What to build

Stories 9–11 (spec-v4 → Server modules → `src/agent/classify.ts`): a **pure** `classify(call, home)`, which returns `{ kind: "allow" }` or `{ kind: "confirm"; reason: "delete" | "secret"; detail }`, with exactly the spec's rules:
- **Bash segmentation:**
  - split on `;`, `&&`, `||`, `|` and newlines;
  - strip `sudo`, `env X=…` and `xargs` prefixes;
  - look at the program word, not a substring, so that `grep rm` and `echo "rm"` stay `allow`.
- **delete:**
  - `rm`, `rmdir`, `unlink`, `trash`, `srm`;
  - `find … -delete`, `find … -exec rm`, `git clean`;
  - `mv … ~/.Trash`, `osascript` text containing `delete`;
  - `mcp__*` tools whose name contains delete, remove or trash.
- **secret:**
  - Read, Grep, Glob or Bash touching `~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.netrc`, `~/.config/gh/hosts.yml`, `~/.claude/.credentials*`;
  - `.env` and `.env.*` (not `.env.example`), `*.pem`, `*.key`, `*.p12`, `id_rsa*`, `id_ed25519*`;
  - `security find-*-password`, `dump-keychain`, `export`;
  - a bare `printenv` or `env`.
- **Fail toward confirm:** `eval`, `bash -c`/`sh -c` or `$(` with any delete or secret token inside → `confirm`.
- **`detail`** is the exact command or path, truncated to 300 chars.
- **`~` and `$HOME` are both recognised,** as is the absolute home path passed in.
- **A module comment** says this is a tripwire against accidents, not a security boundary.

## Red first

A table of **≥ 40 rows**, covering:
- every rule above;
- the look-alikes that must stay `allow`: `git rm --cached`, `grep -r "rm -rf" .`, `echo ~/.ssh`, `cat .env.example`, `ls ~/.Trash`, a non-MCP tool with "remove" in its input;
- the fail-toward-confirm cases.

## Acceptance criteria

- [ ] `classify` is pure: no fs, no env, `home` is an argument.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.
