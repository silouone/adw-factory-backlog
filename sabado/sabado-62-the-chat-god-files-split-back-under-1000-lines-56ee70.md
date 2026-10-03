---
id: sabado-62-the-chat-god-files-split-back-under-1000-lines-56ee70
type: chore
status: in-progress
priority: 1
created: 2026-10-03
depends: [sabado-39-a-turn-that-read-untrusted-text-cannot-write-unconfirmed-3e8f09]
attempts: []
---
# chore(chat): the three chat god files split back under 1 000 lines, and leave the size baseline

> **Why now:** #1333, #1334 and #1336 pushed three files past the size-ratchet
> threshold (`scripts/size-ratchet.py`, R1 of `sabado-23`). Because none of them
> was in `size-baseline.json`, the `front` CI job went red on `main` and on every
> open PR. PR #1354 stopped the bleeding by **recording** them in the baseline
> (`chat_loop.py` 1058, `ai_tools.py` 1004, `ai_tools_writes.py` 1002).
> Recording is a stopgap. This ticket pays the debt.
>
> **Depends on `sabado-39`** (PR #1337), because it edits `chat_loop.py` and
> `ai_tools.py` too. Split after it lands, not under it.

## Requirements

- [ ] **R1** `backend/app/ai/chat_loop.py`, `backend/app/api/ai_tools.py` and
      `backend/app/api/ai_tools_writes.py` are each **under 1 000 lines**.
- [ ] **R2** Their three entries are **removed** from `size-baseline.json`, so
      the ratchet goes back to holding them at the 1 000-line threshold. No other
      baseline entry changes. Do not run `--update-baseline`.
- [ ] **R3** The change is a **pure move**: no behaviour change, no renames of
      public names, and no prompt or tool-catalogue text edited. Every moved
      name is importable from its new module. Update importers to the new home
      rather than re-exporting through the old file, unless a re-export is the
      only way to keep a `monkeypatch.setattr("app.ai.chat_loop.X", …)` target
      meaningful. Check the 6 `monkeypatch.setattr` sites in `backend/tests`
      that patch these modules. A patch on the old path silently stops working
      once the name moves.
- [ ] **R4** Each new module holds one idea and is named for it. Suggested
      seams, not binding:
  - `chat_loop.py`: the untrusted-content marking (`mark_untrusted` …
    `tool_content`, the sentinels) is self-contained.
  - `ai_tools.py`: the `CATALOGUE` table (~275 lines) and the specs derived
    from it.
  - `ai_tools_writes.py`: contacts / members / assets apart from events /
    documents.
- [ ] **R5** No new file crosses 1 000 lines.

## Files

The three files above, their new sibling modules, their importers,
`size-baseline.json`, and `backend/tests/` only where an import path changes.

## Verify

- [ ] `python3 scripts/size-ratchet.py` → `OK — no file grew past its baseline.`
      with the three entries gone from `size-baseline.json`.
- [ ] The backend suite is green with **no test edited except import paths** (R3).
- [ ] `git diff --stat` shows that the lines leaving the three files arrive in
      their new modules (a move, not a rewrite).
