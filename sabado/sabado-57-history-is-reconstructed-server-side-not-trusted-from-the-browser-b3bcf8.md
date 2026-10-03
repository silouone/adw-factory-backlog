---
id: sabado-57-history-is-reconstructed-server-side-not-trusted-from-the-browser-b3bcf8
type: bug
status: blocked
priority: 1
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996]
attempts: [{"runId":"sabado-57-history-is-reconstructed-server-side-not-trusted-from-the-browser-b3bcf8-1791045711290","branch":"adw/sabado-57-history-is-reconstructed-server-side-not-trusted-from-the-browser-b3bcf8","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-57-history-is-reconstructed-server-side-not-trusted-from-the-browser-b3bcf8-1791045711290/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5","stages":{"plan":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"build":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"test":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"build-test-only":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"build-fix":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"revise-test-only":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"repair":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"review-standards":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"review-spec":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"review-fix":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"ci-repair":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"},"rebase-resolve":{"gear":"G2","provider":"claude","model":"claude-sonnet-5-5"}},"blockedAt":{"node":"build-test-only","reason":"ticket \"sabado-57-history-is-reconstructed-server-side-not-trusted-from-the-browser-b3bcf8\" node \"build-test-only\": tool call denied for ticket \"sabado-57-history-is-reconstructed-server-side-not-trusted-from-the-browser-b3bcf8\": Bash — classifier — \"Permission for this action was denied by the Claude Code auto mode classifier. Reason: [Interfere With Workloads]. If you have other tasks that don't depend on this action, continue working on those. IMPORTANT: You *may* attempt to accomplish this action using other tools that might naturally be used to accomplish this goal, e.g. using head instead of cat. But you *should not* attempt to work around this denial in malicious ways, e.g. do not use your ability to run tests to execute non-test actions. You should only try to work around this restriction in reasonable ways that do not attempt to bypass the intent behind this denial. If you believe this capability is essential to complete the user's request, first try a safer method. Get as much of the rest of the task done as you can, then STOP and explain to the user what you were trying to do and why you need this permission. Let the user decide how to proceed. This denial applies to the outcome, not only this exact command: don't pursue the same outcome through another tool, interpreter, host, encoding, sub-agent or later turn, and don't record ways around it. Concretely, these all count as pursuing the same outcome: running the same command in smaller pieces; leaving the flagged part out of this call and covering it in another; reading the same file or data with a different tool (Read, Grep, head, awk, a script); re-issuing it with different quoting, flags, paths or hosts. If this was a batch or range operation, you may re-run it without the flagged items, but do not then act on the flagged items separately — leave those for the user. If this denial names something that would clear it — for example a first-hand read that shows the missing source — doing that is not pursuing the denied outcome: do it, and if it shows what the denial asked for, you may redo the action citing it. To allow this type of action in the future, the user can add a Bash permission rule to their settings.\""}}]
---
# fix(chat): the turn's history is read from the thread the server stored, not from the JSON the browser sent

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 3 (e), first
> bullet — marked `SEC:`. Re-verified on `main` at `c82ba2ec`.

## What happens today

`POST /ai/turn` takes `history` as a **multipart form field containing a JSON
string supplied by the browser**:

```python
history: str = Form(default="[]"),          # backend/app/api/ai.py:640
...
raw_history = json.loads(history or "[]")   # :678
turns = [ChatMessage(**m) for m in raw_history] if isinstance(raw_history, list) else []
```

The last 6 messages with `role in ("user", "assistant")` go straight into the
model's transcript. **An authenticated client can forge assistant turns** — put
words in the assistant's mouth that it never said — and steer the next answer
with them. The thread is already persisted server-side (the `conversations`
table, migration `0070_conversations`); the browser is simply trusted to hand it
back faithfully.

A second, smaller defect rides along: `capture_turn` accepts `conversation_id`
(`chat_feedback.py:112`) and the `/ai/turn` call site passes none, so **every
app-path turn is written with `conversation_id` NULL** despite the FK and the
cascade-erasure guarantee in `chat_turn.py:22-24`.

## The red test

A turn is sent whose client-supplied `history` contains an assistant message that
is **not** in the stored conversation. Assert the message does not reach the
model's transcript. Today it does — that is the red.

> **Waits on `sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996`.**
> This ticket changes what the chat *does*. The chat has **zero end-to-end
> coverage** and `run_turn` is called by no backend test, so nothing today would
> catch a conduct regression. The journey lands first, then this.

## Requirements

- [ ] **R1** The turn's history is read from the server's own `conversations`
      store, scoped to the caller, for the conversation the request names. The
      client's `history` field is no longer the source of truth.
- [ ] **R2** `conversation_id` is passed at the `/ai/turn` capture call site so
      the turn row joins its thread and the cascade erasure means what
      `chat_turn.py:22-24` says it means.
- [ ] **R3** The existing truncation rule is preserved exactly — last 6 messages,
      roles `user`/`assistant`, non-empty content. **This ticket changes where
      history comes from, not how much of it there is.**
- [ ] **R4** A turn on a conversation the caller does not own is refused the same
      way every other cross-household read is (`readable_owner_ids` / the route's
      own 404), not merely filtered to empty.
- [ ] **R5** Backward compatibility is a deliberate, stated choice: say in the PR
      body whether the `history` form field is removed, ignored, or kept for a
      release — and make the front agree with whichever is chosen.

## Files

`backend/app/api/ai.py` · `backend/app/api/conversations.py` (read path only) ·
`frontend/src/pages/dashboard/Conversation.tsx` + `frontend/src/api/ports/ai.ts`
if R5 removes the field · `backend/tests/`.

## Verify

- [ ] The red test above is green after the change.
- [ ] A test asserts a conversation belonging to another account cannot be
      replayed into a turn.
- [ ] A test asserts `ChatTurn.conversation_id` is non-NULL on an app-path turn.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `frontend-check` · `frontend-build` · `test` · `test-frontend` — all green.

## Out of scope

Summarisation, compaction, a token budget or any cross-turn fact memory — all
separate, and `ai_docs/2026-09-30-sabado-synthesis-what-is-next.md` §5 records
that even go1, at 720 files, never built compaction. Raising the 6-message
window.
