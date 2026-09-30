---
id: sabado-35-the-founding-lane-reads-each-document-once-7c3a5c
type: bug
status: in-progress
priority: 1
created: 2026-09-30
caps: {minutes: 180, turns: 600, stallMinutes: 25}
depends: [sabado-32-the-extraction-stage-says-how-long-it-took-b1c44c]
attempts: []
---
# fix(extraction): the founding lane reads each document once, instead of once per anchor

> **Finding:** `ai_docs/2026-09-30-sabado-ingestion-delta.md` §7 (Tier 0 #0),
> from the 2026-09-21 backward pass. **The largest wall-clock win available in
> ingestion: ~40 % of the dominant stage.**
>
> **`depends: sabado-32`** — not for correctness, but because `sabado-32` is
> what makes the improvement *visible*. Landing this first means the 40 % is
> unmeasurable. **Fire it alone**, after 32 is merged: it is the only ticket in
> this wave touching the extraction hot path.
>
> **Shares `household_read.py` with `sabado-32`,** which owns the `cost.timing`
> call sites. This ticket owns `:483-484` and `:684-691`.

## What happens today

`gather_founding_documents_gmail` (`enrich.py:2738`) lists its query **once per
anchor** — `origin/main:enrich.py:2756-2764`:

```python
with gmail_client(mail_account_id, timeout=60) as client:
    gids = _list_ids(client, auth, f"({_FOUNDING_GMAIL_QUERY}){_GMAIL_NOISE_FILTER}", None)
    ...
    seen_hash: set[str] = set()
```

`seen_hash` and the `fetched` gate are **declared inside the call**, so they
dedupe *within* one anchor and cannot see another's results. Then:

- `_founding_lane` (`household_read.py:483`) concatenates per-anchor results
  (`:691`); `anchor in done` (`:684`) only skips anchors from *previous* reads.
- `mails = founding + list(found_mails)` (`:484`) — a plain concatenation.
- `source_messages_from_mails` appends with **no seen-set**
  (`household.py:1463-1468`).
- `_scan_union_pick`'s dedupe (`enrich.py:670-680`) is applied to the union
  only — **the founding mails never pass through it.**

So the same message is fetched, OCR'd and extracted once per anchor that
matched it. Measured on a real mailbox: **700 of 1 659 extraction entries were
repeats of 242 distinct messages**, and **~1 900 of 3 236 model calls** were
duplicates.

Extraction is **49 % of wall clock** on a 147-minute read
(`tickets/findings/2026-09-21-ingestion-under-15-min.md` §5). Removing ~40 % of
it is an order of magnitude more than the entire OCR stage is worth — the OCR
audit put OCR at **under 1 %** of the same run.

## Requirements

- [ ] **R1** One listing for all anchors. Hoist the query out of the per-anchor
      loop, or keep the per-anchor call and dedupe its results against a set
      that **outlives** a single anchor. Either shape is acceptable; the
      invariant is R2.
- [ ] **R2** **A Gmail message id appears at most once** in what `_founding_lane`
      returns, and at most once in `mails = founding + list(found_mails)` — i.e.
      a message reached by both the founding lane and the union is extracted
      once, not twice.
- [ ] **R3** The dedupe key is the Gmail message id. Do not dedupe on a body
      hash (two distinct messages can share a body) and do not dedupe on the
      filename term.
- [ ] **R4** The anchor that matched a message is still recorded, even when the
      message is kept only once. A document must not lose its provenance to the
      dedupe — if a message matched two anchors, that is information, not a
      duplicate to discard silently.
- [ ] **R5** `anchor in done` (`:684`) keeps its current meaning: skipping
      anchors resolved by a **previous** read is a separate mechanism and stays.
- [ ] **R6** No change to the pace, the client's lifetime, or the noise filter.
      The lane keeps its own paced client (`:2756`) for the documented reason.

## Files

`backend/app/extraction/enrich.py` (`gather_founding_documents_gmail` at
`:2738-2790`) · `backend/app/extraction/household_read.py` (`_founding_lane`,
`:483-484` and `:684-691`) · `backend/app/extraction/household.py`
(`source_messages_from_mails`, `:1463-1468`) ·
`backend/tests/test_founding_lane.py`. Nothing else.

## Verify

- [ ] **The red test, first.** Two anchors whose queries both match one Gmail
      message id: assert the message appears **once** in the lane's output and
      once in the messages handed to extraction. It **fails on the current
      tree** — it appears twice. That failure is the ticket.
- [ ] A second red test: a message reached by **both** the founding lane and the
      scan union is extracted once.
- [ ] Regression: a message matching exactly one anchor is unaffected, and the
      per-anchor `seen_hash`/`fetched` gates still refuse a within-anchor repeat.
- [ ] Regression: provenance survives — a message matching two anchors still
      records both (R4).
- [ ] **Quantify it in the PR body.** With `sabado-32` merged,
      `stage_seconds["extraction"]` is available: state the distinct-vs-total
      message count on the test fixture, and the model-call count before and
      after. The audit's numbers to beat are 700/1 659 duplicate entries and
      ~1 900/3 236 duplicate calls.
- [ ] Target gates: `mypy app/` · `lint-imports` · `api-debt-check` ·
      `alembic heads` · `alembic-branch-check` · `test` — all green.

## Out of scope

Merging the two extraction passes (Tier 0 #3). Commit-per-batch (Tier 0 #4).
Parallelising headline chunks (Tier 0 #2). The 48→24-month window (Tier 0 #6).
The Gmail pacer's 5000/4500 tuning (Tier 0 #5). Any `ReadBudget` or wave
machinery — Tier 1, and it needs commit-per-batch first.
