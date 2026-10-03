---
id: hq-43-the-cone-talks-to-the-companion-8f9c7b
type: feat
status: queued
priority: 2
depends: [hq-42-ask-is-a-guarded-streaming-route-7370f6]
created: 2026-10-03
caps: {minutes: 180, turns: 450}
attempts: []
---
# The cone talks to the companion: streamed reply, activity line, approval card, resumable session

> Spec: `~/personal_project/silou-hq/docs/spec-v4-companion.md` (binding; it amends rule #1 of `CLAUDE.md`, and `docs/spec-v1.md` + `docs/spec-v2.md` (incl. amendment v2.1) + `docs/spec-v3-mail-calendar.md` still bind otherwise). Rules: `CLAUDE.md`. Claude-specific code lives ONLY in `src/agent/runtime/claude.ts`. No test spawns the real `claude` binary.

## What to build

Stories 2–8 and 11 on the client (spec-v4 → Client):
- **The seam grows, runtime-agnostically:**
  - `ask` may yield `activity`, `approval` and `session`;
  - `AgentAdapter` gains `respond(id, allow)`;
  - `none` is unchanged and still honest.
- **`createServerAdapter(fetch)`:**
  - POSTs `/ask` (`content-type: application/json`, attachments as base64);
  - parses NDJSON across split chunks;
  - aborts on the session signal;
  - `respond` POSTs `/ask/approve`.
- **Adapter choice at mount** comes from `GET /ask.json`.
- **The session:**
  - it keeps the `session` id and sends it as `resume` on follow-ups while the modal is open; `leave()` aborts and forgets it;
  - pending approvals are queued in order.
- **The approval card,** inside the cone over the reply:
  - a reason badge (`DELETE` ember, `SECRET` gold), the tool, the detail in monospace, and **Allow** / **Refuse**;
  - the state is `listening` while it shows.
- **The activity line** is one muted line under the reply, cleared on `done`.
- **The session id** is small, under the reply, with copy.
- **🔊** speaks the final reply via the existing v2.1 path, once, after `done`, never per delta.

## Red first

- **Server adapter:**
  - NDJSON split mid-line across chunks;
  - the abort cancels the fetch;
  - `respond` posts the right body.
- **Session:**
  - `resume` is sent on the second ask and dropped after `leave()`;
  - two approvals queue in order.
- **Approval card:**
  - it renders reason, tool and detail;
  - Allow and Refuse call `respond`;
  - it shows while state = `listening`.
- **Empty and offline:** `/ask.json` failing → the `none` adapter and its honest message.

## Acceptance criteria

- [ ] `grep -ri claude src/web` finds nothing.
- [ ] `bun run lint && bunx tsc --noEmit && bun test` green.

## Manual checks (operator)

- [ ] "what's queued in the factory?" streams a real answer with an activity line.
- [ ] `rm /tmp/hq-throwaway` shows a DELETE card; Refuse keeps the file.
- [ ] `cat ~/.ssh/config` shows a SECRET card.
- [ ] Esc mid-run stops it (no new `tool` ledger lines).
- [ ] 🔊 speaks once, in your voice.
- [ ] `claude --resume <id>` in a terminal continues it.
