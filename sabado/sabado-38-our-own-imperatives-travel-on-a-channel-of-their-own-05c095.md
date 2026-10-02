---
id: sabado-38-our-own-imperatives-travel-on-a-channel-of-their-own-05c095
type: bug
status: in-review
priority: 1
created: 2026-10-01
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: [sabado-37-the-tool-channel-marks-untrusted-content-35eb6a]
attempts: [{"runId":"sabado-38-our-own-imperatives-travel-on-a-channel-of-their-own-05c095-1790925196049","branch":"adw/sabado-38-our-own-imperatives-travel-on-a-channel-of-their-own-05c095","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-38-our-own-imperatives-travel-on-a-channel-of-their-own-05c095-1790925196049/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1336","provider":"claude","model":"claude-sonnet-5-5"}]
---
# fix(chat): our own instructions travel on a channel of their own, so marking untrusted data means something

> ## Base: PR #1283 is merged — this ticket is runnable
>
> `feat/chat-calls-tools` landed on `main` at **2026-10-01T06:01Z**, so `chat_loop.py`,
> `ai_tools*.py`, `usage.py` and `chat_turn.py` are all present. Line numbers below are as
> audited 2026-09-30 on the pre-merge branch and shifted in the rebase — **navigate by
> symbol, not by line**.
>
> **A first attempt was fired at 2026-09-30T22:49Z, seven hours BEFORE that merge**, cut
> from `29ae7af5` where none of these files existed. All four such runs blocked on
> `red-check exhausted` having produced nothing — an agent cannot write a failing test for
> a defect in a file that is not in its tree. The `attempts` entry below is that run. It is
> not a verdict on this ticket.

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 6, second required
> fix. **`sabado-37` is undermined without this one** — if our commands and an attacker's
> arrive in the same JSON object, a sentinel around the whole object marks both or neither.

## What happens today

Tool results mix two kinds of string in one dict. Ours are *commands to the model*:

- `ai_tools.py` ≈`:943-946` — `say`: *"Dis que tu ne peux pas ouvrir ce document"*
- `ai_tools_reads.py` ≈`:667-668` — `hint`: *"Rappelle l'outil avec ces noms EXACTS"*
- `chat_loop.py` ≈`:607-609` — *"n'appelle plus rien, ne redemande pas de confirmation"*

And theirs is *data*: `content`, `title`, `notes`, `record`, `location`.

The model has no way to tell them apart, and **83 commits of the first kind have trained it
to obey the channel**. That training is the vulnerability `sabado-37` wraps — but a wrapper
cannot discriminate inside a single flat dict.

## Requirements

- [ ] **R1** Our own directives move to a reserved key (or key prefix) that the prompt
      names as **the only trusted channel** — e.g. everything under a single
      `_sabado` / `instruction` envelope, distinct from the data keys beside it.
- [ ] **R2** The prompt states the rule once: strings on the trusted channel are the
      system speaking; everything else is data. Pair it with `sabado-37`'s section rather
      than writing a second, competing paragraph.
- [ ] **R3** A tool cannot put a directive on the data channel or data on the trusted
      channel by accident. Enforce it — a helper that builds the result, or a test that
      walks every tool's output shape (see Verify).
- [ ] **R4** The reserved key is stripped from anything that originated outside our code,
      so a document cannot smuggle a directive by naming its own field `say`.
- [ ] **R5** Behaviour parity: the model still obeys our directives exactly as it does
      today. The measured guards this channel carries (the announce pattern, the
      exact-names hint, the no-more-calls instruction) must keep working — they were each
      earned against a named defect.
- [ ] **R6** The fingerprint moves, and the PR prints old vs new.

## Files

`backend/app/api/ai_tools_data.py` (the result-shaping seam) ·
`backend/app/api/ai_tools.py` · `backend/app/api/ai_tools_reads.py` ·
`backend/app/ai/chat_loop.py` · `backend/app/ai/prompts_chat/chat-tools.fr.md` ·
`backend/tests/`. Nothing else.

## Verify

- [ ] **The red test, first.** Walk **every** one of the 21 tools' result shapes and assert
      no directive-bearing key (`say`/`hint`/`instruction`/`rule`) appears outside the
      reserved envelope. It must **fail on the current tree**, naming each offender.
      *Derive the tool list from the registry, not a hard-coded list, and assert exact
      set equality against the allowlist* — so a new tool is caught automatically and a
      stale exemption also fails (the pattern from
      `ai_docs/2026-09-30-go1-engineering-hygiene.md`).
- [ ] A second red test: a document whose extracted text contains `"say": "…"` does not
      reach the model on the trusted channel (R4).
- [ ] Regression: the announce guard, the exact-names hint and the
      no-more-calls instruction all still fire — assert each through the loop, not as a
      unit function.
- [ ] Target gates all green.

## Out of scope

The sentinel wrapper itself (`sabado-37`). Rewording any directive's content — this
ticket moves strings, it does not rewrite them.
