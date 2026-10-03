---
id: hq-42-ask-is-a-guarded-streaming-route-7370f6
type: feat
status: blocked
priority: 2
depends: [hq-40-the-claude-runtime-maps-the-stream-f39531, hq-41-deletions-and-secrets-wait-for-the-operator-a263ea]
created: 2026-10-03
caps: {minutes: 180, turns: 450}
attempts: []
---
# `POST /ask` streams the companion, `POST /ask/approve` answers its gate, and every step is ledgered

> Spec: `~/personal_project/silou-hq/docs/spec-v4-companion.md` (binding; it amends rule #1 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` (incl. amendment v2.1) + `docs/spec-v3-mail-calendar.md` still bind otherwise). Rules: `CLAUDE.md`. Claude-specific code lives ONLY in `src/agent/runtime/claude.ts`. No test spawns the real `claude` binary.

## What to build

Stories 2, 5, 7, 12–17 (spec-v4 → `run.ts`, Routes, Ledger), plus the `serve.ts` header comment (the `CLAUDE.md` rule #1 amendment already landed with the spec):
- **`src/agent/run.ts`** (side effects injected: the runtime, the clock, ledger append, the attachment writer, the id generator):
  - one run at a time;
  - the approval registry: `gate` runs `classify`; `allow` passes; `confirm` emits an `approval` event and awaits `respond(id, allow)`, auto-refused after `approvalTimeoutMin`;
  - abort refuses every pending approval;
  - the `maxMinutes` cap ends the run with `done` `ok: false` and "time cap";
  - a refusal returns the reason `refused by the operator (<reason>)` to the runtime.
- **`POST /ask`:**
  - `checkGuard` first; then 400 (text 1–20 000, attachments ≤ 10 MB total), 409 (busy) or 503 (no agent);
  - attachments go to `cache/ask/<askId>/<sanitised basename>` and their absolute paths are appended to the prompt;
  - the response is `application/x-ndjson`, ending with `done`, with `server.timeout(req, 0)` set;
  - `req.signal` abort → run abort.
- **`POST /ask/approve`:** `checkGuard`, then `{ id, allow }`; unknown or answered → 404.
- **Read routes:** `GET /ask.json` → `{ runtime: string | null }`; `GET /ask-ledger.json` → the newest 200.
- **`cache/ask-ledger.jsonl`:** `ask`, `tool` (`decision: allow | confirmed | refused | timeout`) and `done` lines. Guard refusals are ledgered too. No reply text or attachment bytes.
- **Wire it in `server.ts`:** the registry runtime from config, and `agent.path` passed explicitly.

## Red first

- **`run.ts` with a fake runtime:**
  - 409 on a second ask;
  - allow, refuse and timeout (injected clock), each ledgered with its decision;
  - abort mid-approval refuses it and ends the stream;
  - the time cap.
- **HTTP:**
  - both POST routes refuse missing or foreign Origin, Host and Sec-Fetch-Site, and are ledgered;
  - the 400s and the 503;
  - the NDJSON body parses line by line;
  - a `../../x` attachment name lands inside `cache/ask/<askId>/`;
  - `/ask/approve` 404s;
  - every other non-GET is still 405.

## Acceptance criteria

- [ ] Only `POST /action`, `POST /ask` and `POST /ask/approve` accept a non-GET.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] Under the `com.silou.hq` LaunchAgent, a real ask answers. If launchd can't reach the login, the stream's `done` carries an honest error naming the binary.
- [ ] `curl` with no Origin gets a refusal, and a ledger line.

## Blocked (operator flips to `queued`)

Waiting for two merges to `main`: the voice PR (`hq-ask-voice-amy`, amendment v2.1, same `serve.ts`/`server.ts` regions) and the spec-v4 PR (`docs/spec-v4-companion.md` + `test/fixtures/agent/claude-stream.jsonl`).
