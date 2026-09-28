---
id: adw-pr-03-ci-and-validation-on-the-card
type: feat
status: queued
priority: 2
created: 2026-09-17
caps: {minutes: 180, turns: 600}
depends: [adw-pr-02-one-github-read-per-target]
attempts: []
---
# CI and validation on the card — and four different ways of saying nothing

Spec: `specs/adw-v1.9-pr-state.md` (Stage 3). Stage 1 built the derivations,
Stage 2 fed them. This is the only stage the operator sees.

## The problem, restated at the point of delivery

The card's last word is *"the PR opened"*. Every slot on it —
lane, mini-Gantt, `green`/`blocked`, cost, time, tokens — is a fact about the
**factory**, never about the **artifact**. So a run that ended `green` and was
then closed by a reviewer renders identically to one that was merged.

Two chips fix that. They answer different questions and **either may
legitimately be empty**.

```
┌─────────────────────────────────────────┐
│ adw-fe-19-instant-nodes-stack…   12m ago│
│ bug · red test first, then the fix      │
│ ▁▃▅▂ mini-Gantt ▁▂▅▃                    │
│ ✓ success  ●●●●●●                       │
│ ◉ CI green   ⧗ awaiting review   #61    │  ← this row
│ $ ~2.98   ◷ 38m 30s   ◈ 4.58M           │
└─────────────────────────────────────────┘
```

## 1. The CI chip

Stage 1's classifier produces **five** states: `passing · failing · cancelled ·
pending · no-checks`.

- [ ] All five are distinguishable, but **only two carry colour.** Green and
      red are what you scan the board for. `cancelled`, `pending` and
      `no-checks` all mean *"no verdict"* and render as one neutral glyph
      whose **tooltip names which of the three it is**. Five facts, two visual
      weights — settled by operator grilling 2026-09-17.
- [ ] **`no-checks` is not green** and `cancelled` is not red.
- [ ] A red chip carries the **failing check's name** and links to its job.
- [ ] A red chip carries the **failing check's name** and links to its job via
      `detailsUrl`, so the operator lands on the log instead of hunting
      through Actions.
- [ ] Follows the existing KPI tooltip idiom (`renderKpi`) — this card already
      has a vocabulary; do not invent a second one.

## 2. The validation chip

Two GitHub fields, rendered as one chip, **independent of CI**:

| Source | Value | Chip |
|---|---|---|
| `state` | `MERGED` | ✓ validated |
| `state` | `CLOSED` | ✕ rejected — closure *is* rejection (`sync-pr-state`'s own rule) |
| `state` | `OPEN` | ⧗ awaiting review |
| `reviewDecision` | `APPROVED` | ✓ approved |
| `reviewDecision` | `CHANGES_REQUESTED` | ⚑ changes requested |
| `reviewDecision` | `REVIEW_REQUIRED` | ⧗ review required |
| `reviewDecision` | `""` | **nothing rendered** |

- [ ] An `OPEN` PR with a `reviewDecision` shows the **verdict**, which is more
      specific than "awaiting"; `MERGED`/`CLOSED` outrank it.
- [ ] **Merged without approval gets a marker.** Measured 2026-09-17 on
      `CoorpAcademy/api-content`: **13 of 88** merged PRs carry
      `REVIEW_REQUIRED`, not `APPROVED` — admin override, or a review
      dismissed by a later push. `state` and `reviewDecision` genuinely
      disagree in production. Render `✓ merged` as the headline with a **small
      superscript dot**, explanation in the tooltip. Not tooltip-only (hides
      an Art. IV audit signal behind a hover nobody thinks to try) and not a
      colour (makes a routine admin merge look like an error).
- [ ] **No such marker on `CLOSED`.** A closed PR is already a rejection;
      "closed without approval" carries no extra fact.
- [ ] `reviewDecision: ""` renders **nothing** — not "unknown", not a spinner,
      not a placeholder. Measured: **16 of 16** PRs on `adw-factory` and
      `clens` are `""`, because they are solo repos. On the 7 Coorpacademy
      targets review is mandatory and this field is the whole point. **Both
      must look right.**
- [ ] The PR number links to the PR.

## 3. Four ways of saying nothing, and they are not the same

This is the requirement most likely to be flattened into one grey dash. Don't.
`card.ts` already draws exactly this distinction with `captured` vs
`noCapture`, and for the same reason: **an honest absence must never look like
a broken one.**

- [ ] **No PR** — the run never reached `open-pr` (blocked, or still live).
      A permanent, correct absence.
- [ ] **No repo identity** — no `github` field and no clone, so nothing is
      resolvable. Permanent until config changes.
- [ ] **gh unavailable** — logged out, offline, rate-limited. Transient, and
      the **only** one that earns a notice.
- [ ] **Not fetched yet** — first paint before the first TTL fill. Transient,
      resolves within seconds, and must not flash a misleading verdict.
- [ ] A card with no PR renders **no chip row at all** — not an empty row.

## 4. Live over SSE, with no new plumbing

- [ ] The chips update over SSE without a reload. `server.ts` already diffs the
      rendered fragment per connection and pushes only on change — a TTL
      refresh changes the chip HTML, the diff fires, the push happens. This is
      the same mechanism that flips a liveness badge after 60 s of silence.
      **Verify it; do not add an event type for it.**

## 5. Which PR belongs to the card

- [ ] A card is a **ticket**, not a run (`TicketCard {ticketId, latest,
      attempts}`). The chip shows **`latest`'s** PR.
- [ ] "Older attempt merged, newer attempt open" is a real state. The attempt
      dots keep their existing meaning; do not overload them.

## 6. Leave room for the factory's own verdict

At the time of writing, `src/pipeline/lanes/shared.ts` has **uncommitted** work
splicing `makeReviewGateLoop` between `gates` and `commit`. When that lands,
runs journal `node-end` events with `details.kind === "review"` and the card
gains the *factory's* two-axis verdict — from the journal, no GitHub call.

That is a **different signal** from the human's: it answers "did the code meet
our standards before anyone looked", not "did a person accept it". They can
disagree, and merging them into one "reviewed" chip destroys exactly what the
disagreement tells you.

- [ ] Lay the chip row out so a third chip is an **addition, not a redesign**.
- [ ] At pickup, re-check `grep -c '"kind":"review"' runs/*/journal.jsonl`. If
      verdicts are now on disk, raise whether both axes should ship together —
      that is a spec question, so stop and ask rather than deciding in code.

## TDD (Art. I — non-negotiable)

Tests first, red, reviewed, then green. Per `adw-v1.2` Decision 3, **the
browser is never a test seam** — if a behaviour cannot be asserted at the
projection or on the rendered HTML string, it is presentation and this ticket
does not specify it. Assert on the HTML `renderTicketCard` returns.

The four absence cases each need their own test and must produce four
**distinguishable** outputs. A test that only checks "no verdict text appears"
passes for all four and proves nothing.

## Verify

- [ ] Every new suite green; four distinct absence renders asserted.
- [ ] Live: the board shows CI and validation for every banked run with a PR,
      and those values match `gh pr view <n>` by hand for three spot-checked
      cards.
- [ ] Live: a card whose run blocked before `open-pr` shows no chip row.
- [ ] Live: with the board open, a TTL refresh pushes a changed chip over SSE
      with no reload.
- [ ] `bun run lint && bunx tsc --noEmit && bun test`.

## Out of scope

The run screen and the v1.7 run panel — board card only in v1.9. Review
feedback **text** (needs a second per-PR `gh pr view`; deferred). Any mutation
— no approving, merging, closing or re-running CI from the board; read-only was
an explicit operator decision and is not reopened here. A full per-check list
on the card face; the failing check's name in a tooltip is the bound.
