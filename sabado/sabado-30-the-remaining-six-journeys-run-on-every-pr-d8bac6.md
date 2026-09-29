---
id: sabado-30-the-remaining-six-journeys-run-on-every-pr-d8bac6
type: feat
status: in-progress
priority: 2
created: 2026-09-28
caps: {minutes: 240, turns: 800, stallMinutes: 25}
depends: [sabado-29-every-pr-walks-a-real-browser-through-a-real-backend-a98ee9]
attempts: [{"runId":"sabado-30-the-remaining-six-journeys-run-on-every-pr-d8bac6-1790678790207","branch":"adw/sabado-30-the-remaining-six-journeys-run-on-every-pr-d8bac6","workspace":"/Users/silouane/personal_project/adw-factory/runs/sabado-30-the-remaining-six-journeys-run-on-every-pr-d8bac6-1790678790207/workspace","outcome":"in-review","pr":"https://github.com/App-sabado/sabado/pull/1318","provider":"claude","model":"claude-sonnet-5-5"}]
---
# test(e2e): the remaining six journeys of the 2026-09-19 audit run on every PR

> `sabado-26` R3's journeys 3–8, verbatim, against the CI stack `sabado-29`
> stood up rather than a deployed slot. Journeys 1–2 landed with `sabado-29`.
> Nothing here is new scope — it is the audit's own list (axis H,
> `.claude/reports/audit-work/H.md`), moved down one tier so it gates PRs.

## Requirements

Each journey is its own spec file under `frontend/e2e/journeys/`, uses the
`sabado-28` account fixture, and **never calls `page.route`**.

- [ ] **R1 — journey 3, declarative onboarding.**
      `03-onboarding.spec.ts`. « Pourquoi » → foyer → logement → « Passer » on
      mail → complete, with the bag data of
      `.claude/personas/sacs/lea-tom-primo-mobile.yaml` (already prefixed).
      Records exist via `GET /children` and `GET /assets`.
- [ ] **R2 — journey 4, documents.** `04-documents.spec.ts`. Upload
      `.claude/fixtures/01_CNI_SPECIMEN.pdf` on `/documents`, open it, attach it
      to the foyer member; the row renders, `GET /shared-documents` lists it,
      « Reliée à » names the member.
- [ ] **R3 — journey 5, the vault is zero-knowledge.** `05-vault.spec.ts`.
      `/vault/setup` passphrase → seal a note → lock → unlock → read it back. A
      network capture over the **whole** journey asserts that **only `*_enc`
      fields leave the browser**, and `beforeunload` wipes the key. This is the
      journey with the highest value and the highest cost to write: a leak here
      is a real privacy failure, and it is not detectable from a DOM test.
- [ ] **R4 — journey 6, circles.** `06-circles.spec.ts`. Create → invite
      `qa+e2e@example.com` → invitation listed → cancel; preview-as-them shows
      nothing sealed (`test_circles.py` semantics, on the real stack).
- [ ] **R5 — journey 7, the calendar round trip.** `07-calendar-feed.spec.ts`.
      Create an event → month view → `GET /calendar/feed?token=…` returns an ICS
      containing it → import that same ICS URL as a source
      (`calendar_share.py`) — a self-generated feed, no real external calendar.
- [ ] **R6 — journey 8, Google SSO start only.** `08-google-start.spec.ts`.
      `/auth/google/available` → click « Continuer avec Google » → the redirect
      targets `accounts.google.com` with the run's own `redirect_uri`
      (`GOOGLE-OAUTH.md`). The **callback** stays covered by `test_google.py`;
      this journey stops at the redirect.
- [ ] **R7 — the budget holds, or it is reported.** Six more journeys on the
      `e2e` job. Target **under 8 minutes total** for the job, the same ceiling
      `sabado-29` R8 set. If the six push it past that, **report the measured
      duration in the PR body and change nothing else** — trimming a journey to
      fit a budget is the operator's decision, not the run's.

## Files

`frontend/e2e/journeys/03-onboarding.spec.ts` …
`08-google-start.spec.ts` (new, six files).

## Verify

- [ ] All six green in the `e2e` job; `grep -rn "page.route"
      frontend/e2e/journeys/` → no match.
- [ ] **Journey 5's capture contains no cleartext note body** — asserted on the
      captured request bodies, not on a screenshot.
- [ ] Journey 7's ICS actually contains the event created in the same run
      (match on its UID or summary, not on a non-empty response).
- [ ] The throwaway account is gone after the job, on a failing run too.
- [ ] The job's measured wall-clock is recorded in the PR body.
- [ ] Target gates: `mypy app/` · `lint-imports` · `alembic heads` ·
      frontend check · frontend build · `test-e2e-boot` · `just test` — green.

## Out of scope

Mail connect and ingestion — no throwaway IMAP mailbox exists, and the persona
guardrail exists precisely because the QA account aliases a real Gmail.
« Parler à Sabado » — no AI provider is configured in CI. Visual regression.
Firefox and WebKit. All four were `sabado-26`'s exclusions and stay excluded.
