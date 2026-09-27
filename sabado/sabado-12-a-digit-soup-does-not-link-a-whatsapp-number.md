---
id: sabado-12-a-digit-soup-does-not-link-a-whatsapp-number
type: bug
status: done
priority: 1
created: 2026-09-19
review: false
caps: {minutes: 150, turns: 450, stallMinutes: 25}
depends: [sabado-00-the-test-loop-is-trustworthy-and-runs-side-by-side]
attempts: [{"runId":"sabado-12-a-digit-soup-does-not-link-a-whatsapp-number-1789850667706","branch":"adw/sabado-12-a-digit-soup-does-not-link-a-whatsapp-number","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-12-a-digit-soup-does-not-link-a-whatsapp-number-1789850667706/workspace","outcome":"blocked"},{"runId":"sabado-12-a-digit-soup-does-not-link-a-whatsapp-number-1789859135141","branch":"adw/sabado-12-a-digit-soup-does-not-link-a-whatsapp-number","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-12-a-digit-soup-does-not-link-a-whatsapp-number-1789859135141/workspace","outcome":"in-review","provider":"claude","model":"sonnet","pr":"https://github.com/App-sabado/sabado/pull/1222"}]
---
# fix(whatsapp): a message full of digits cannot brute-force another user's link code

> **Audit:** **P1-4** (axis J), re-verified by the synthesis
> (`service.py:110-124` reads `link.code in digits` — substring). Held out
> of the LANES batch as "security"; included because it is a one-hour,
> single-file cross-user read. `review: false`: hard-gated.

## What happens today

`backend/app/whatsapp/service.py:110-124`: `digits = re.sub(r"\D", "", text)`;
every unverified link with a live 6-digit code is loaded; the match is
`link.code in digits` — a **substring** test; there is no per-`wa_id`
attempt cap. `api/whatsapp.py:33,125`: 6-digit code, 15-minute TTL.

One 4 096-digit message tests ~4 091 codes. With N users mid-linking,
~245/N messages are expected to succeed. On success the attacker's number is
verified on the victim's account and `_answer_question` (`service.py:263-274`)
answers over `build_chat_context(user)` — household, calendar, budget.

## Requirements

- [ ] **R1** Exact match: the message, digits only, must **equal** a live
      code (`digits == link.code`).
- [ ] **R2** Attempt cap per sender: Redis key `rl:wa:<wa_id>`, 5 attempts
      per 15 minutes, through the existing `check_rate_limit` helper (the
      auth routes already use it). Over the cap, the message is ignored
      and nothing is linked.
- [ ] **R3** Nothing else changes: the 6-digit code, the TTL, the success
      path, the reply texts.

## Files

`backend/app/whatsapp/service.py` · `backend/tests/test_whatsapp.py`.

## Verify

- [ ] Red test: `test_whatsapp.py::test_a_digit_soup_containing_a_valid_code_does_not_link`
      — a live link with code `123456`; an inbound message whose digits are
      `9912345600`; assert the link is still unverified. RED today.
- [ ] Red test: `test_whatsapp.py::test_the_sixth_wrong_code_in_fifteen_minutes_is_ignored`
      — five wrong exact codes then the right one from the same `wa_id`;
      assert not linked. RED today.
- [ ] The existing linking tests in `test_whatsapp.py` stay green.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · `just test` — all green.

## Out of scope

8-digit codes, binding the link to `WHATSAPP_BOT_NUMBER`, the webhook's own
rate limit, moving WhatsApp processing to the worker (K P2).
