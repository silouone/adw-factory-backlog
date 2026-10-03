# silou-hq — v4 spec: the companion behind ASK HQ

> **Status:** decisions taken by the operator 2026-10-03.
> **Amends:** rule #1 of `CLAUDE.md` (see Amendment). Everything in `docs/spec-v1.md`, as amended by
> `docs/spec-v2.md` (including amendment v2.1, `GET /speech`) and `docs/spec-v3-mail-calendar.md`, that
> this spec does not amend still binds.
> It fills the "ASK HQ agent backend" item that spec-v2's Out of Scope deferred to "its own spec".
> **Labels:** `ready-for-agent` once it is on `main`.
> **Sources:**
> - Two probes run on 2026-10-03 against Claude Code 2.1.288 (`~/.local/bin/claude`) and
>   `@anthropic-ai/claude-agent-sdk` 0.3.287. The CLI probe's stream is committed as
>   `test/fixtures/agent/claude-stream.jsonl`.
> - The operator round of 2026-10-03.

## Problem Statement

The ASK HQ cone works (spec-v2), but the only adapter behind it is `none`, which says "No agent is
connected to ASK HQ yet." The operator wants a real agent there. It should not be a builder or a
narrow assistant. It should be **their companion**: one agent with the operator's own Claude Code
setup that can search and change anything on the laptop. That includes orchestrating silou HQ and
the adw factory, and driving macOS applications.

## Operator decisions (2026-10-03)

| # | Question | Decision |
|---|---|---|
| D1 | Which runtime answers | **Claude Code**, the operator's install (`~/.local/bin/claude`), with their default model. Codex and ollama come later, behind the same server-side adapter. |
| D2 | What it can reach | **Everything on the laptop.** The working directory is `~`, and it has full tool access with no permission prompts, including Bash, so `osascript`, `open` and `shortcuts` can drive macOS apps. |
| D3 | Whose setup it uses | **The operator's.** User settings, `~/.claude/CLAUDE.md`, skills, agents and MCP servers load as they do in a terminal. There are two exceptions, both forced: the output style is `default` and settings hooks are off (see "No double voice"). |
| D4 | What needs the operator's OK | **File deletion and secret access** wait for an explicit Allow in the cone. Everything else runs unprompted. |
| D5 | Delivery | A spec plus factory tickets (hq-40..43). |
| D6 | May it trigger HQ verbs | It does not call `POST /action`; the origin guard refuses it anyway. With full Bash it can act directly, and that is recorded in the ask ledger. |
| D7 | Speech-to-text | **Still out of scope.** The 🎙 mic drives the listening visual only. |

## Solution

The client keeps one runtime-agnostic adapter that streams from a new server route. On the server,
a `claude` runtime adapter drives Claude Code through the Agent SDK. The SDK is used rather than
`claude -p`, because only an in-process `PreToolUse` callback can **pause a tool call while the
operator decides**. A pure classifier flags deletions and secret reads. A flagged call becomes an
approval card in the cone, and the run waits for Allow or Refuse.

Probe facts this design rests on (2026-10-03):
- With `permissionMode: "bypassPermissions"` and `--settings {"outputStyle":"default","disableAllHooks":true}`,
  an SDK `PreToolUse` callback still fires. It awaited 3 s, returned `permissionDecision: "deny"` on
  `rm /tmp/…/victim.txt`, and the file survived. `init` reported `output_style: "default"`, and no
  settings hooks ran.
- OAuth from the operator's login works through the system binary
  (`pathToClaudeCodeExecutable`). The `claude` binary bundled with the SDK is **not** used; an
  earlier one was found stale and denied.
- A trivial two-turn ask took about 7 s and reported `total_cost_usd` ≈ 0.29 on the default model.
- `stream_event` / `content_block_delta` / `text_delta` carries the token stream (use
  `includePartialMessages`). `assistant` messages carry `tool_use` blocks, and `result` carries
  `session_id`, `is_error`, `result` and `total_cost_usd`.
- **Subagents are gated too.** When the companion delegated `rm` to an `Agent` subagent, the same
  callback saw the `Agent` call and then the subagent's `Bash` call (with `agent_id` set), denied it,
  and the file survived.
- With user MCP servers loaded, a reply can end with notes about servers that failed to connect.
  That is accepted (D3); it's the operator's setup.

## User Stories

**Asking**
1. As the operator, I want ASK HQ answered by my own Claude Code, so that the companion knows my CLAUDE.md, skills and tools.
2. As the operator, I want the reply to stream into the cone as it is written, so that a long answer shows progress.
3. As the operator, I want a one-line activity readout of the tool running now (`Bash · ls ~/adw/backlog`), so that I see what the companion is doing between words.
4. As the operator, I want follow-up questions in the same open modal to continue the same conversation, so that I don't repeat context.
5. As the operator, I want closing the modal (✕ or Esc) to stop the run at once, with no further tool calls, so that nothing keeps acting after I leave.
6. As the operator, I want the session id shown under the reply, so that I can continue it in a terminal with `claude --resume <id>`.
7. As the operator, I want attached files handed to the companion as paths it can read, so that I can drop a screenshot or a log on the centre and ask about it.
8. As the operator, I want 🔊 to speak the reply through `GET /speech`, as v2.1 already does, so that the companion talks in my voice.

**The approval layer**
9. As the operator, I want any file deletion to wait for my Allow, so that the companion never deletes by accident.
10. As the operator, I want any read of a secret to wait for my Allow, so that keys never reach a transcript unseen.
11. As the operator, I want the approval card to show the reason (`delete` or `secret`), the tool and the exact command or path, so that I decide on the real thing.
12. As the operator, I want a refusal to reach the companion as a refusal with a reason, so that it continues without that step instead of retrying blind.
13. As the operator, I want an unanswered approval refused after 5 minutes, so that a forgotten card never hangs a run forever.

**Safety and record**
14. As the operator, I want only HQ's own page to be able to start an ask or answer an approval, so that another site in my browser can never drive my companion.
15. As the operator, I want every ask, every tool call and every approval decision ledgered, so that "what did HQ change on my Mac?" (v1 story 58) still has an answer.
16. As the operator, I want one ask at a time, so that two runs never race over the same files.
17. As the operator, I want an honest error in the cone when the runtime can't start (not logged in, binary missing, the LaunchAgent can't reach the Keychain), so that a missing source is never faked.
18. As the factory's builder, I want everything Claude-specific in one server-side runtime module, so that codex or ollama can be added without touching the shell or the route.

## Amendment to rule #1 (lands with this spec, before any v4 ticket is dispatched)

The current rule #1 opens: "Writes are local, allowlisted and ledgered. Exactly one write route
exists: `POST /action`. Every other non-GET returns 405. An action names a node id the graph recorded
and a verb from that node kind's fixed list. The client never sends a path, a shell string or a command."

It is replaced by:

> 1. **Writes are local, guarded and ledgered.** Exactly three write routes exist: `POST /action`,
>    `POST /ask` and `POST /ask/approve`, all origin-guarded by `checkGuard`. Every other non-GET
>    returns 405. An action names a node id the graph recorded and a verb from that node kind's fixed
>    list, and the client never sends it a path, a shell string or a command. **`POST /ask` is the
>    one exception:** it carries free text to the operator's companion agent, which has full access by
>    the operator's decision (spec-v4 D2). Deletions and secret reads wait for the operator's Allow.
>    Every ask, tool call and approval is appended to `cache/ask-ledger.jsonl`. [rest of rule #1 unchanged]

This spec supersedes spec-v2's Out of Scope item "ASK HQ agent backend", except for speech-to-text.
spec-v2's text is left as written, and `CLAUDE.md`'s binding line lists v4 after v2.

The `CLAUDE.md` edit is committed with this spec (`spec-v4-companion` PR), not by a ticket.

`CLAUDE.md` "Any LLM is a tool" is kept: Claude-specific code lives only in `src/agent/runtime/claude.ts`.

## Implementation Decisions

### Config (`hq.config.json`)

```jsonc
"agent": {
  "runtime": "claude",                     // a key of the server-side runtime registry
  "claude": { "bin": "~/.local/bin/claude", "model": null },   // null = the operator's default
  "cwd": "~",
  "path": "/opt/homebrew/bin:~/.local/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin",
  "maxMinutes": 30,
  "approvalTimeoutMin": 5
}
```

- `agent` absent → the client gets the `none` adapter, and spec-v2 story 71 still holds.
- Parsing fails fast and names the field: an unknown runtime, or a non-absolute path after `~`
  expansion.
- `path` is the agent's `PATH`, passed **explicitly in the child's env**. Bun 1.2.4 ignores
  runtime `PATH` changes for its own spawn resolution, and launchd's default `PATH` is
  `/usr/bin:/bin:/usr/sbin:/sbin`.

### Server modules

- **`src/agent/types.ts`** — runtime-agnostic types:
  ```ts
  type AgentEvent =
    | { state: AgentState }
    | { markdown: string }                       // cumulative: the whole reply so far
    | { activity: string }                       // one line, ≤ 120 chars
    | { approval: { id: string; reason: "delete" | "secret"; tool: string; detail: string } }
    | { session: string }
    | { done: { ok: boolean; error?: string; costUsd?: number } };
  interface Runtime {
    readonly id: string;
    ask(q: { text: string; files: readonly string[]; resume?: string },
        gate: (call: ToolCall) => Promise<{ allow: true } | { allow: false; reason: string }>,
        signal: AbortSignal): AsyncIterable<AgentEvent>;
  }
  type ToolCall = { tool: string; input: Readonly<Record<string, unknown>> };
  ```
- **`src/agent/runtime/claude.ts`** — the only Claude-specific module:
  - **pure `claudeOptions(config, q)`**, which builds the SDK options: `cwd`,
    `pathToClaudeCodeExecutable`, `permissionMode: "bypassPermissions"`,
    `allowDangerouslySkipPermissions: true`, `settingSources: ["user"]`,
    `extraArgs.settings = {"outputStyle":"default","disableAllHooks":true}`,
    `includePartialMessages: true`, `resume`, `model` when set, `env` with `PATH`, and
    `appendSystemPrompt` from `src/agent/companion-prompt.md`;
  - **pure `mapMessages`**, which turns SDK messages into `AgentEvent`s. Each message type has one
    source, so nothing is counted twice:
    - `markdown` comes **only** from `stream_event` `text_delta`s, accumulated. A new text block
      after a tool call is joined with `\n\n`.
    - `activity` comes **only** from the `tool_use` blocks of `assistant` messages. Their text blocks
      are ignored, because the deltas already carried them.
    - Messages with a non-null `parent_tool_use_id` (a subagent's own stream) produce no
      `markdown`; their `tool_use` blocks still produce `activity`.
    - `init` and `result` become `session`. `result` becomes `done` (`is_error` → `ok: false` with
      the result text).
  - the edge: `query()` with a `PreToolUse` callback (timeout 600 s) that awaits `gate` and returns
    `permissionDecision: "deny"` with the reason on refusal. `signal` aborts the SDK's
    `abortController`.
- **`src/agent/classify.ts`** — the pure approval classifier, `classify(call, home) →
  { kind: "allow" } | { kind: "confirm"; reason: "delete" | "secret"; detail: string }`:
  - **delete:**
    - a Bash command with a segment (split on `;`, `&&`, `||`, `|`, newlines, after stripping
      `sudo`, `env X=…`, `xargs`) whose program is `rm`, `rmdir`, `unlink`, `trash` or `srm`;
    - `find … -delete`, `find … -exec rm`, `git clean`;
    - `mv … ~/.Trash`, and `osascript` text containing `delete`;
    - any MCP tool (`mcp__*`) whose name contains `delete`, `remove` or `trash`;
  - **secret:**
    - a Read, Grep, Glob or Bash call that touches `~/.ssh`, `~/.aws`, `~/.gnupg`, `~/.netrc`,
      `~/.config/gh/hosts.yml` or `~/.claude/.credentials*`;
    - any `.env` or `.env.*` file (not `.env.example`), `*.pem`, `*.key`, `*.p12`, `id_rsa*` or
      `id_ed25519*`;
    - `security find-*-password`, `security dump-keychain`, `security export`;
    - `printenv` or `env` with no command after it;
  - **fail toward confirm:** a Bash command containing `eval`, `bash -c`/`sh -c`, or `$(`… whose
    text contains any delete or secret token is `confirm`;
  - **detail** is the exact command or path, truncated to 300 chars.
  - **This is a tripwire against accidents, not a security boundary.** A full-access agent can
    write a script that deletes. The spec states that and does not try to close it.
- **`src/agent/run.ts`** — the ask orchestrator (side effects injected): one run at a time, the
  approval registry (`id → resolver`, auto-refused after `approvalTimeoutMin`, all refused on
  abort), the `maxMinutes` cap and ledger appends.
- **`src/agent/companion-prompt.md`** — appended to Claude Code's system prompt. It says:
  - you are the operator's companion, reached through silou HQ's ASK HQ cone;
  - the reply is rendered in a small round area and may be spoken, so lead with a short answer of
    one to three sentences, then details;
  - HQ's read routes are on `http://127.0.0.1:<port>` (`/graph.json`, `/factory.json`,
    `/ledger.json`), and the factory's web is at `$ADW_WEB`;
  - deletions and secret reads go through an operator approval, and a refusal is final for this ask;
  - factory writes (dispatching, flipping ticket status, merging PRs) happen only when the operator
    asks for them in this ask (see O1).

- **The SDK is pinned** to `@anthropic-ai/claude-agent-sdk@0.3.287`, the version the probes ran on.
  Bumping it means re-capturing the fixture.

### Routes

- **`POST /ask`** — `checkGuard` first, then a JSON body
  `{ text: string; resume?: string; attachments?: { name: string; base64: string }[] }`:
  - `text` is 1–20 000 chars, and attachments total ≤ 10 MB. Otherwise → 400.
  - A run already in progress → 409. `agent` not configured → 503.
  - Attachments are written to `cache/ask/<askId>/<basename>` (the basename is sanitised to
    `[A-Za-z0-9._-]`), and their absolute paths are appended to the prompt.
  - The response is `application/x-ndjson`, one `AgentEvent` per line, ending with `done`.
  - The idle timeout is lifted for this request (`server.timeout(req, 0)`). The `maxMinutes`
    cap bounds the run instead.
  - `req.signal` aborting (the client left) aborts the run: no further tool call starts, and
    pending approvals are refused.
- **`POST /ask/approve`** — `checkGuard`, then `{ id: string; allow: boolean }`. An unknown or
  already-answered id → 404. The answer resolves the gate.
- **`GET /ask.json`** → `{ runtime: string | null }`, so the page knows whether an agent is
  configured.

### Ledger (`cache/ask-ledger.jsonl`)

Lines are appended, one per `{ t, askId, kind, … }`:
- `ask` (text truncated to 200, the attachment names, `resume`);
- `tool` (tool, detail truncated to 300, `decision: allow | confirmed | refused | timeout`);
- `done` (ok, error, ms, costUsd).

Attachment bytes and reply text are not ledgered. A `GET /ask-ledger.json` route (newest 200)
mirrors `/ledger.json`.

### Client (`src/web/ask/`)

- **The seam grows by two event kinds,** and stays runtime-agnostic:
  - `ask` yields `activity`, `approval` and `session` alongside `state` and `markdown`;
  - the adapter gains `respond(id: string, allow: boolean): Promise<void>`;
  - `none` keeps working unchanged.
- **`createServerAdapter(fetch)`** posts to `/ask` with `content-type: application/json`, reads the
  NDJSON stream line by line, and aborts the fetch on the session's signal.
- **Choosing the adapter.** At mount, the page reads `GET /ask.json` → `{ runtime: string | null }`.
  `null` or a failed read gives the `none` adapter; a runtime id gives the server adapter.
- **The session** keeps the `session` id for follow-ups while the modal stays open. `leave()`
  aborts and forgets it.
- **The approval card** sits inside the cone over the reply. It shows a reason badge (`DELETE` in
  ember, `SECRET` in gold), the tool, the detail in monospace, and **Allow** / **Refuse**. While it
  shows, the state is `listening`, so the dust cap turns gold. More than one pending approval is
  queued in order.
- **The activity line** is one muted line under the reply, replaced on each `activity`, and cleared
  on `done`.
- **The session id** goes under the reply, small, with a copy button.

### No double voice

The operator's output style tells Claude Code to run `higgs_tts.py` itself, and the operator's
hooks speak too. Both are forced off for the companion (`outputStyle: "default"`,
`disableAllHooks: true`; probe-verified), so the only voice is HQ's `GET /speech`.

## Testing Decisions

- **The same two seams as v1 and v2, plus the runtime seam.** Fixtures go in, and events, HTTP
  responses or ledger lines come out.
- **`classify`** is table-tested: at least 40 rows covering every rule above, plus look-alikes that
  must stay `allow`:
  - `rm` inside a string or a grep pattern;
  - `.env.example`, `~/.ssh` mentioned in an echo, `git rm --cached`;
  - `remove` in a non-MCP tool name.
- **`mapMessages`** is driven by `test/fixtures/agent/claude-stream.jsonl` (real, captured
  2026-10-03): the cumulative markdown at each step, one activity per `tool_use`, and `session` and
  `done` from `result`. A hand-made `is_error` fixture covers the failure path.
- **`claudeOptions`** is snapshot-tested on a config: the forced output style, the disabled hooks,
  bypass mode, the explicit `PATH`, and `resume` only when given.
- **The run orchestrator, with a fake `Runtime`:**
  - 409 on a second ask;
  - approval allow, refuse and timeout (injected clock);
  - abort refuses pending approvals and stops the stream;
  - the `maxMinutes` cap;
  - the ledger lines for each.
- **HTTP:**
  - `/ask` and `/ask/approve` refused without guard headers, and ledgered;
  - 400 on empty or oversized text and attachments;
  - 503 with `agent` absent;
  - an NDJSON body that parses line by line;
  - attachments land in `cache/ask/<askId>/` with sanitised names, and a `../` name can't escape.
- **The client:**
  - the server adapter parses split NDJSON chunks;
  - the approval card renders, and Allow / Refuse calls `respond`;
  - `leave()` aborts the fetch;
  - the `none` adapter is unchanged.
- **No test spawns the real `claude`.** The real runtime is checked manually (below).
- **Gates:** `bun run lint && bunx tsc --noEmit && bun test`.
- **Manual checks the factory can't run** (operator):
  - a real ask under the `com.silou.hq` LaunchAgent: auth works from launchd's context, and if it
    doesn't, the cone shows an honest error;
  - `rm` on a throwaway file shows a DELETE card, and Refuse keeps the file;
  - `cat ~/.ssh/config` shows a SECRET card;
  - Esc mid-run stops it (no new ledger `tool` lines);
  - 🔊 speaks once, in the operator's voice;
  - `claude --resume <id>` in a terminal continues the session.

## Out of Scope

- Speech-to-text, and voice in (D7).
- Other runtimes (codex, ollama). The registry has one entry.
- A conversation transcript in the cone. It still shows one reply, as spec-v2 has it.
- HQ verbs called by the agent through `/action`.
- Approval for anything other than deletion and secrets (for example `git push --force` or
  `launchctl`). That is a later amendment if the operator wants it.
- Remote access: HQ stays on `127.0.0.1`.
- Cost caps beyond `maxMinutes`. The cost is ledgered per ask.

## Open decisions

- **O1 — the companion and the supervisor's single-writer rule.** The supervisor direction
  (`adw-factory/ai_docs/2026-10-01-supervisor-and-hq-continuation.md`) makes single-writer a hard
  rule for factory state. The companion can write that state through Bash (the `adw` CLI, the ticket
  store, `gh`). v4 bounds this only through the prompt: factory writes happen when the operator asks
  in the ask. When the supervisor lands, a later amendment decides whether the companion goes
  through it or becomes it.

## Further Notes

- **Order.** The voice PR (`hq-ask-voice-amy`, amendment v2.1) touches the same `serve.ts` and
  `server.ts` regions, so it merges first, then this spec, then hq-40/41 (parallel), hq-42, hq-43.
- **Sessions persist** in Claude Code's own store (`~/.claude/projects/-Users-silouane/`), as for a
  terminal session in `~`. HQ stores no transcript.
