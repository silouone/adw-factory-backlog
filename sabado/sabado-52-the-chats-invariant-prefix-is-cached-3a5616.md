---
id: sabado-52-the-chats-invariant-prefix-is-cached-3a5616
type: feat
status: queued
priority: 2
created: 2026-10-02
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996]
attempts: []
---
# feat(chat): the turn's invariant prefix is cached, so a round stops paying for the same 8 500 tokens

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 7 (d) fourth
> bullet + the quantified paragraph; `ai_docs/2026-09-30-sabado-chat-state.md` §6.
> Re-verified on `main` at `c82ba2ec`.
>
> **Frame this as latency and round budget, not cost.** Total measured Mistral
> spend across the bench's entire history is **$2.588**. The prize is a p95 of
> 5.33 s and a round the loop can afford to spend on `completeness` — the judge's
> worst dimension at **3.54**.

## What happens today

Three facts compose into a guaranteed miss:

1. **`MistralProvider.chat` sends `tools` + `tool_choice="auto"` and sets no
   cache directive at all.** It relies on Mistral's implicit per-node prefix
   cache, whose own module docstring (`backend/app/ai/provider.py:16-19`)
   measures the prize at *"9 067 billed tokens on a miss against 43 on a hit —
   140×"*, at a hit rate of roughly one call in two.
2. **`ClaudeProvider` sets `cache_control: {"type": "ephemeral"}` on the system
   block** (`provider.py:222-228`) — and then **raises `NotImplementedError` the
   moment `tools` is non-empty** (`:262`). The tool loop never sees it.
3. **The last round deliberately swaps the system prompt** — `system =
   system_prompt(catalogue, names)` at `chat_loop.py:781`, replacing the
   `with_list=False` prefix built at `:749`. That guarantees a full prefix miss
   on the final round of **every** turn.

Quantified: a static system prompt of 20 481 bytes (~6k tokens) plus a spec
payload the code itself sizes at 7 509 characters (~2.5k tokens), re-sent 3 to 5
times per turn. **~8.5k tokens of byte-identical prefix, re-sent 3–5×.**

> **Waits on `sabado-44-a-conversation-is-walked-in-a-real-browser-on-every-pr-5ba996`.**
> This ticket changes what the chat *does*. The chat has **zero end-to-end
> coverage** and `run_turn` is called by no backend test, so nothing today would
> catch a conduct regression. The journey lands first, then this.

## Requirements

- [ ] **R1** An explicit cache directive on the path that actually runs — the
      Mistral tool path. Follow `_MistralCachePrefixHook` (`provider.py:248-297`),
      which already stamps `prompt_cache_key` on every chat completion; **extend
      it, do not write a second mechanism beside it.**
- [ ] **R2** The invariant prefix and the volatile remainder are **separated**:
      the monolith + tool specs in the cached slot, anything per-turn or per-date
      outside it. Note that `today_hint()` rotates daily by construction — say in
      the PR body which side of the boundary you put it on and why.
- [ ] **R3** **Stop swapping the system prompt on the last round.** Append the
      closed list as a **suffix** instead (`chat_loop.py:781`), so the prefix is
      stable across all rounds of a turn. The last round is still served with no
      tools — that behaviour does not change.
- [ ] **R4** The hit rate is **observable**. `cached_tokens` is already read back
      and stored; surface it per turn so the next person can tell a working cache
      from a believed one.
- [ ] **R5** Not a behaviour change. The same prompt content reaches the model,
      in the same order, on every round. A fingerprint change is expected and
      fine; a *conduct* change is a bug.

## Files

`backend/app/ai/provider.py` · `backend/app/ai/chat_loop.py` ·
`backend/tests/`.

## Verify

- [ ] A test asserts the system prefix sent on round 1 and round 3 of the same
      turn are **byte-identical**.
- [ ] A test asserts the last round no longer rebuilds the system prompt.
- [ ] **State the measured before/after in the PR body** — median prompt tokens
      per turn, or `cached_tokens` share, from whatever run you can make. If no
      live key is available, say so rather than quoting the estimate as a result.
- [ ] Target gates: `backend-typecheck` · `backend-imports` · `alembic heads` ·
      `test` — all green.

## Out of scope

Response caching (memoising answers). Switching provider. Touching the
`/ai/analyze` or WhatsApp paths, which have their own prompt and their own
hook. Any change to `MAX_ROUNDS`.
