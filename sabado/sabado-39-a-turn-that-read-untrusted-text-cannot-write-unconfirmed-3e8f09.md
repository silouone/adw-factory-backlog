---
id: sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09
type: bug
status: in-review
priority: 1
created: 2026-10-01
caps: {minutes: 150, turns: 500, stallMinutes: 25}
depends: [sabado-37-the-tool-channel-marks-untrusted-content-35eb6a]
attempts: [{"runId":"sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09-1790925211826","branch":"adw/sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09-1790925211826/workspace","outcome":"in-review","provider":"claude","model":"claude-sonnet-5-5","pr":"https://github.com/App-sabado/sabado/pull/1337","ciRounds":0},{"runId":"sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09-ci-1790977866092","branch":"adw/sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09-1790925211826/workspace","outcome":"blocked","provider":"claude","model":"claude-sonnet-5-5"}]
---
# fix(chat): a turn that read a document cannot write to the household without a confirmation

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

> **Finding:** `ai_docs/2026-09-30-sabado-pr1283-seven-axes.md` Axis 6, third required
> fix — described there as **"the cheapest real mitigation"**. It is the defence that
> works even if the marking of `sabado-37` is imperfect or a future tool forgets it.

## What happens today

Thirteen write tools exist. **Three gate** — `delete_event`, `delete_contact`,
`delete_document`, via `confirms=` (`ai_tools.py` ≈`:929-975`). **Ten do not:**
`create_event`, `update_event`, `create_expense`, `create_asset`, `create_member`,
`create_contact`, `update_contact`, `update_member`, `update_asset`, `attach_document`.

`update_asset`, `update_member` and `update_contact` are silent field overwrites on the
household's identity and finance records — the audit's phrasing — and they are exactly the
tools reachable from the attacker path in `sabado-37`.

Worse, `_carry_the_removal` (`chat_loop.py` ≈`:569-610`): on a turn whose **question**
contains "supprime", a read returning exactly one row **auto-opens a deletion window with
the preview already fetched** — so the confirmation modal is reachable without the model
choosing to delete.

## Requirements

- [ ] **R1** A turn is **tainted** once it has received attacker-influenceable content —
      `get_documents` content, an imported calendar event's free text, or the attached
      document block. Taint is a property of the **turn**, not of the tool call.
- [ ] **R2** On a tainted turn, **any** write requires the existing confirmation flow.
      Reuse `confirms=` and the `/pending` redemption — do **not** build a second
      confirmation mechanism. (`sabado-40` hardens that flow; this ticket only widens who
      uses it.)
- [ ] **R3** An untainted turn is **unchanged**. A person saying *"ajoute le dentiste
      jeudi"* still writes in one turn with no modal. This must not become a confirmation
      dialog on every write — that would train the person to click through, which is worse
      than no gate.
- [ ] **R4** The confirmation the person sees says **why** it is being asked on a tainted
      turn, and it is distinguishable from a deletion confirmation. A modal with no reason
      is a modal that gets dismissed.
- [ ] **R5** `_carry_the_removal` does not auto-open a window on a tainted turn.
- [ ] **R6** Taint is recorded on the turn capture, so a later reader can tell which turns
      were tainted and whether the gate fired. (If `sabado-43`'s round rows exist by then,
      it belongs in `policy_flags`; if not, on the turn.)

## Files

`backend/app/ai/chat_loop.py` (taint tracking, `_carry_the_removal`) ·
`backend/app/api/ai_tools.py` (the `confirms=` gate) ·
`backend/app/ai/prompts_chat/chat-tools.fr.md` (if the model must be told) ·
`frontend/src/pages/dashboard/` (the reason in the modal, R4) ·
`backend/tests/`. Nothing else.

## Verify

- [ ] **The red test, first.** A turn that calls `get_documents` and then `update_asset`
      asserts the write **did not execute** and a confirmation is pending instead. It must
      **fail on the current tree** — today it writes.
- [ ] A second red test: `_carry_the_removal` does not auto-open on a tainted turn.
- [ ] **Regression, and this is the one that matters most:** a turn that reads **no**
      untrusted content writes in one turn with no modal — assert it for at least three of
      the ten currently-unconfirmed writes.
- [ ] Measure the cost in the PR body: over the bench corpus, what share of turns become
      tainted? If it is most of them, R3 has failed and the taint definition is too broad.
- [ ] Target gates all green.

## Out of scope

`/pending` idempotency and expiry (`sabado-40`). Confirming writes on untainted turns —
deliberately not doing that. The sealed-field registry.
